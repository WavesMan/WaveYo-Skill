# Python SSH Audit Pattern — WaveYo 风格

> 触发条件：需要编写 Linux 批量巡检或修复工具时加载。
> 来源证据：cve-2026-31431-fleet-remediator 的 SSH 批量操作模式。

---

## 核心架构：audit / fix 模式分离

```
audit 模式（只读）           fix 模式（写入）
    │                            │
    ├─ SSH 连接                  ├─ SSH 连接
    ├─ 只读命令                  ├─ 修复命令
    ├─ 状态采集                  ├─ 状态变更
    ├─ 风险评估                  ├─ 结果验证
    └─ 报告输出                  └─ 修复报告
```

---

## SSH 执行器

```python
import paramiko
import threading
from queue import Queue
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
from typing import Optional

@dataclass
class HostResult:
    host: str
    success: bool
    output: str
    error: Optional[str] = None
    risk_level: str = "unknown"  # ok / warning / critical

class SSHExecutor:
    def __init__(self, max_workers: int = 10, timeout: int = 30):
        self.max_workers = max_workers
        self.timeout = timeout

    def execute(self, hosts: list[str], command: str,
                username: str, key_file: str) -> list[HostResult]:
        results = []
        with ThreadPoolExecutor(max_workers=self.max_workers) as executor:
            futures = {
                executor.submit(
                    self._ssh_exec, host, command, username, key_file
                ): host
                for host in hosts
            }
            for future in as_completed(futures):
                host = futures[future]
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    results.append(HostResult(
                        host=host, success=False, output="", error=str(e)
                    ))
        return results

    def _ssh_exec(self, host: str, command: str,
                  username: str, key_file: str) -> HostResult:
        client = paramiko.SSHClient()
        client.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        try:
            client.connect(
                hostname=host, username=username,
                key_filename=key_file, timeout=self.timeout
            )
            stdin, stdout, stderr = client.exec_command(command)
            output = stdout.read().decode()
            error = stderr.read().decode()
            return HostResult(
                host=host, success=True,
                output=output, error=error or None
            )
        except Exception as e:
            return HostResult(
                host=host, success=False, output="", error=str(e)
            )
        finally:
            client.close()
```

---

## Audit 模式（只读巡检）

```python
class AuditRunner:
    def __init__(self, executor: SSHExecutor):
        self.executor = executor

    def audit(self, hosts: list[str], checks: list[dict],
              username: str, key_file: str) -> dict:
        """
        只读检查所有主机状态。
        返回：{host: {check_name: HostResult}}
        """
        report = {}
        for check in checks:
            name = check["name"]
            command = check["command"]
            print(f"[audit] 正在检查: {name} ({len(hosts)} 台主机)")

            results = self.executor.execute(
                hosts, command, username, key_file
            )

            for r in results:
                if r.host not in report:
                    report[r.host] = {}
                # 风险评估
                r.risk_level = self._assess_risk(name, r)
                report[r.host][name] = r

        return report

    def _assess_risk(self, check_name: str, result: HostResult) -> str:
        if not result.success:
            return "critical"  # 连接失败
        # 根据检查项评估风险
        if "CVE" in check_name or "vulnerable" in result.output.lower():
            return "critical"
        if "warning" in result.output.lower():
            return "warning"
        return "ok"
```

---

## Fix 模式（批量修复）

```python
class FixRunner:
    def __init__(self, executor: SSHExecutor):
        self.executor = executor
        self.fix_history = []  # 记录修复操作

    def fix(self, hosts: list[str], fix_commands: list[dict],
            username: str, key_file: str,
            dry_run: bool = False) -> list[HostResult]:
        """
        执行修复操作。dry_run=True 时只打印不执行。
        """
        results = []
        for cmd in fix_commands:
            name = cmd["name"]
            command = cmd["command"]

            if dry_run:
                print(f"[fix] DRY RUN: {name} → {command[:80]}...")
                continue

            print(f"[fix] 正在执行: {name} ({len(hosts)} 台主机)")
            step_results = self.executor.execute(
                hosts, command, username, key_file
            )
            results.extend(step_results)

            # 记录修复历史
            for r in step_results:
                self.fix_history.append({
                    "host": r.host,
                    "command": name,
                    "success": r.success,
                    "output": r.output[:500],
                })

        return results
```

---

## 报告输出（多格式）

```python
import json
import csv

class ReportGenerator:
    @staticmethod
    def to_json(results: dict, output_path: str):
        with open(output_path, "w") as f:
            json.dump(results, f, indent=2, default=str)

    @staticmethod
    def to_csv(results: list[HostResult], output_path: str):
        with open(output_path, "w", newline="") as f:
            writer = csv.DictWriter(f, fieldnames=[
                "host", "success", "risk_level", "error"
            ])
            writer.writeheader()
            for r in results:
                writer.writerow({
                    "host": r.host,
                    "success": r.success,
                    "risk_level": r.risk_level,
                    "error": r.error or "",
                })

    @staticmethod
    def to_markdown(results: list[HostResult]) -> str:
        lines = ["# Audit Report", "", "| Host | Status | Risk | Error |",
                  "|------|--------|------|-------|"]
        for r in results:
            lines.append(f"| {r.host} | {'✅' if r.success else '❌'} "
                         f"| {r.risk_level} | {r.error or '-'} |")
        return "\n".join(lines)

    @staticmethod
    def summary(results: list[HostResult]) -> dict:
        total = len(results)
        success = sum(1 for r in results if r.success)
        critical = sum(1 for r in results if r.risk_level == "critical")
        return {
            "total": total,
            "success": success,
            "failed": total - success,
            "critical": critical,
        }
```

---

## 进度心跳

```python
import time
import threading

def progress_heartbeat(stop_event: threading.Event, interval: float = 5.0):
    """长时间操作输出进度心跳"""
    start = time.time()
    while not stop_event.is_set():
        elapsed = time.time() - start
        print(f"[进度] 已运行 {elapsed:.0f}s ...")
        time.sleep(interval)
    print(f"[进度] 完成，总耗时 {time.time() - start:.0f}s")

# 使用方式
stop = threading.Event()
heartbeat_thread = threading.Thread(target=progress_heartbeat, args=(stop,))
heartbeat_thread.start()

# ... 执行长时间操作 ...
results = executor.execute(hosts, command, username, key_file)

stop.set()
heartbeat_thread.join()
```

---

## 注意事项

- audit 模式不修改目标主机，fix 模式需显式 `--fix` 标志
- SSH 连接超时默认 30s，大批量场景可调大
- ThreadPoolExecutor worker 数不超过 20（避免 SSH 连接风暴）
- 失败主机按错误类型分组，在报告末输出失败分类摘要
- 正式场景建议泛化具体 CVE 编号为"Linux 主机批量巡检与修复工具"
- 修复操作记录完整历史，支持回滚检查

> 来源：WaveYo / cve-2026-31431-fleet-remediator 的模式提炼。
> 正式使用时建议泛化 CVE 编号。
