 文件结构 (13个文件, ~1880行):

  ai-code-reviewer/
  ├── SKILL.md (252行)                    # 主技能文件, 4阶段工作流
  └── references/
      ├── severity-guide.md (73行)        # Critical/High/Medium/Low 分级标准
      ├── output-template.md (81行)       # 结构化报告模板
      ├── hallucinated-apis.md (158行)    # 类别1: 幻觉API检测
      ├── pattern-inconsistency.md (117行)# 类别2: 模式不一致检测
      ├── copy-paste-variations.md (138行)# 类别3: 复制粘贴变体检测
      ├── missing-edge-cases.md (172行)   # 类别4: 缺失边界检查
      ├── logical-flaws.md (178行)        # 类别5: 逻辑缺陷检测
      ├── missing-tests.md (118行)        # 类别6: 缺失测试检测
      ├── placeholder-abandonment.md (136行)# 类别7: 占位符遗弃检测
      ├── type-system-abuse.md (160行)    # 类别8: 类型系统滥用检测
      ├── security-naive-patterns.md (162行)# 类别9: 安全模式检测
      └── redundant-logic.md (134行)      # 类别10: 冗余逻辑检测

  核心工作流: 输入处理 → 加载代码库上下文（提取项目规范、依赖、工具库清单）→ 10个类别的AI特征检测 → 交叉引用去重 →
  生成结构化审查报告

支持的触发短语包括: 「review this PR for AI artifacts」「check if this was
  AI-generated」「find AI code issues」「audit this diff for hallucinated APIs」「AI code review」等。
