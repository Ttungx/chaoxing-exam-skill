# chaoxing-exam-skill

超星学习通（Chaoxing）考试自动答题 Skill，适用于 QoderWork / Claude Code 等 AI Agent 平台。

## 适用场景

本 Skill 适合用于**未开启防作弊检测的考试**，例如：

- 通识课 / 水课的在线考试（如思政课、公选课等）
- 教师创建的练习性测试卷
- 自建试卷的预览与调试

**不适用于**开启了以下防护措施的考试：
- 切屏检测 / 锁屏监控
- 人脸识别 / 活体检测
- 摄像头全程录像
- IP/设备变更锁定

请自行判断使用风险，遵守所在学校的相关规定。

## 功能特性

- 支持**整卷预览**、**逐题作答**、**练习**三种考试模式
- 自动提取题目文本（单选/多选/填空/程序题）
- 批量选择答案并 AJAX 保存到服务器
- 作答状态验证与视觉状态恢复

## 安装

### 通过 QoderWork

在 QoderWork 中搜索  技能并安装，或将本仓库的  复制到  目录下。

### 通过 Claude Code Marketplace

如果本仓库已注册到 marketplace，可使用：

Installing plugin "chaoxing-exam"...

## 使用方法

1. 在浏览器中打开超星学习通考试页面并登录
2. 对 AI Agent 说："帮我答这个超星的考试"
3. Agent 会自动激活本 Skill，按流程完成作答

## 文件结构



## 技术要点

超星考试页面的选项不是标准 HTML radio/checkbox，而是自定义 div + onclick 事件。本 Skill 的核心技术发现：

- 显示字母与内部  值随机映射，必须按显示字母定位
- 单选用  而非 （后者 AJAX 可能失败）
- 多选用 
- JavaScript 执行必须用 IIFE 包裹才能返回值
-  提交后视觉状态可能丢失，需恢复  类

## License

MIT
