# Colleague-Skill 5 层 Persona 结构（参考提取）

来源：https://github.com/titanwings/colleague-skill/dot-skill/prompts/persona_builder.md

## 框架

```
Layer 0：核心规则（硬规则，任何情况下不可违背）
Layer 1：身份（定位、角色、标签）
Layer 2：表达风格（口头禅、说话方式、场景示例）
Layer 3：决策判断（优先级、推进/回避、处理质疑）
Layer 4：人际行为（对上级/下级/平级、压力下行为）
Layer 5：边界雷区（不喜欢、会拒绝、回避的话题）
```

## 关键设计原则

1. **Layer 0 的质量决定整个 Persona 的质量**
2. 每条规则必须是可执行的"在什么情况下会怎么做"，不能是形容词
3. Layer 2 要有真实感——直接写他会说的话
4. 所有结论必须有原材料支撑（或标注"基于XX推断"）
5. Correction 层优先于所有其他层

## 适配到工程人格

由于无聊天数据，将：
- Layer 0：性格标签 → 工程原则（从项目习惯推断）
- Layer 1：公司/职级 → 技术定位
- Layer 2：口头禅/说语 → 工程表达习惯
- Layer 3：人际决策 → 技术决策模式
- Layer 4：人际行为 → 协作行为
- Layer 5：雷区 → 技术红线（从边界文件提取）
