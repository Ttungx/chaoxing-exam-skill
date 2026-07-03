---
name: chaoxing-exam
description: "Automate answering exams on Chaoxing (超星学习通) platform. Handles exam preview (整卷预览), per-question (逐题作答), and practice (练习) modes on mooc1.chaoxing.com. Use when the user wants to auto-fill, batch-answer, or complete exam questions on the Chaoxing learning platform, or mentions 超星/学习通/考试/答题."
version: 1.0.0
---

# 超星学习通自动答题

自动化操作超星学习通(mooc1.chaoxing.com)考试页面，支持整卷预览、逐题作答、练习等多种模式。本 skill 仅负责技术操作流程，答案内容需 agent 根据题目自行推理判断。

## 前置要求

- 用户已在浏览器中打开考试页面并登录
- Browser MCP 工具可用（`mcp__builtin_browser__*`）

## 核心工作流

```
Task Progress:
- [ ] Step 1: 识别考试模式
- [ ] Step 2: 提取题目内容
- [ ] Step 3: 分析 DOM 结构，构建题目映射
- [ ] Step 4: 推理答案并选择选项
- [ ] Step 5: 提交保存答案
- [ ] Step 6: 验证作答状态
```

### Step 1: 识别考试模式

通过 URL 和页面标题判断：

| 模式 | URL 特征 | 页面标题 | 特点 |
|------|----------|----------|------|
| 整卷预览 | `/exam-ans/mooc2/exam/preview` | "整卷预览" | 所有题目在同一页面，可一次性作答 |
| 逐题作答 | `/exam-ans/` 其他路径 | "考试" | 每题一页，需逐题导航 |
| 练习模式 | `/exercise/` | "练习" | 类似考试但无计时 |

**优先处理整卷预览模式**（效率最高）。逐题模式下需要额外处理翻页导航。

### Step 2: 提取题目内容

使用 `get_page_text` 获取全页文本：

```
tool: mcp__builtin_browser__get_page_text
params: { tabId, max_chars: 200000 }
```

注意：`read_page`（无障碍树）只能看到结构但看不到题目文本，**必须用 `get_page_text`**。

提取后解析出：题目编号、题型（单选/多选/填空/程序）、题干、选项。

### Step 3: 分析 DOM 结构

**这是最关键的一步。** 超星的选项不是标准 `<input type="radio">`，而是自定义 div + onclick 事件。

#### 3.1 定位题目元素

```javascript
var qs = document.querySelectorAll('.questionLi');
// 每个 questionLi 的 data 属性 = 题目 ID
var qid = q.getAttribute('data');
```

#### 3.2 题目类型判断

```javascript
var typeName = q.querySelector('input[name^=typeName]').value;
// "单选题" | "多选题" | "填空题" | "程序题" 等
```

#### 3.3 选项结构

选项是 `<div class="clearfix answerBg">` 元素，内含：

```html
<div class="clearfix answerBg" onclick="saveSingleSelect(this,'qid');">
    <span data="内部值" qid="题目ID" class="saveSingleSelect num_option fl">A</span>
    <div class="fl answer_p">选项文本</div>
</div>
```

**关键概念——显示字母 vs 内部值：**
- `span.textContent`（如 "A"）= 用户看到的显示字母
- `span.data` 属性（如 "B"）= 提交给服务器的内部值
- **两者不对应，每题随机映射！** 必须按显示字母定位选项。

单选 span 类名：`saveSingleSelect` / `num_option`
多选 span 类名：`saveMultiSelect` / `num_option_dx`

答案存储位置：`<input id="answer{qid}">`（单选 name=answer，多选 name=answers）

完整 DOM 细节见 [reference.md](reference.md)。

### Step 4: 选择答案

#### 4.1 JavaScript 执行注意事项

**必须用 IIFE 包裹代码**，否则多语句脚本返回 `undefined`：

```javascript
(function(){ /* code */ return 'result'; })()
```

#### 4.2 单选题作答

**不要直接调用 `saveSingleSelect`**（会触发异步 AJAX 可能失败），改用底层 `addChoice`：

```javascript
(function(){
  var answers = {1:'B', 2:'D', 3:'A', ...};  // 题号:显示字母
  var qs = document.querySelectorAll('.questionLi');
  var results = [];
  for(var i=0; i<20; i++){
    var q = qs[i], letter = answers[i+1];
    var opts = q.querySelectorAll('.answerBg');
    for(var j=0; j<opts.length; j++){
      var span = opts[j].querySelector('.num_option');
      if(span && span.textContent.trim() === letter){
        addChoice($(opts[j]).find('.saveSingleSelect'));
        results.push((i+1)+':'+letter);
        break;
      }
    }
  }
  return results.join(', ');
})()
```

`addChoice($span)` 会：给 span 添加 `check_answer` 类 + 更新隐藏 input 值。

#### 4.3 多选题作答

多选题用 `clickSaveMultiSelect`，支持 toggle：

```javascript
(function(){
  var answers = {21:['B','C','E'], 22:['A','B','D','E'], ...};
  var qs = document.querySelectorAll('.questionLi');
  var results = [];
  for(var i=startIdx; i<endIdx; i++){
    var q = qs[i], selected = answers[i+1];
    var opts = q.querySelectorAll('.answerBg');
    var clicked = [];
    for(var j=0; j<opts.length; j++){
      var letter = ['A','B','C','D','E'][j];
      if(selected.indexOf(letter) >= 0){
        clickSaveMultiSelect(opts[j], q.getAttribute('data'));
        clicked.push(letter);
      }
    }
    results.push((i+1)+':'+clicked.join(''));
  }
  return results.join(', ');
})()
```

#### 4.4 修改答案

重新调用 `addChoice`（单选 toggle）或 `clickSaveMultiSelect`（多选 toggle）即可切换。

如需清除重选：
```javascript
$('.choice'+qid).removeClass('check_answer');
$('#answer'+qid).val('');
// 然后重新选择正确选项
```

### Step 5: 提交保存

全部选完后调用 `submitForm` 将答案 AJAX 保存到服务器：

```javascript
(function(){
  submitForm(true, null, function(){});
  return 'submitted';
})()
```

参数说明：`submitForm(tempSave=true, quietSubmit=null, callBack)`

**注意：提交后 `check_answer` 视觉状态可能丢失**，但隐藏 input 值仍在。如需恢复：

```javascript
(function(){
  for(var i=0; i<40; i++){
    var q = document.querySelectorAll('.questionLi')[i];
    var qid = q.getAttribute('data');
    var val = $('#answer'+qid).val();
    if(val){
      q.querySelectorAll('.answerBg').forEach(function(opt){
        var s = $(opt).find('.num_option, .num_option_dx');
        if(val.indexOf(s.attr('data')) >= 0) s.addClass('check_answer');
      });
    }
  }
  return 'restored';
})()
```

### Step 6: 验证作答状态

检查所有隐藏 input 是否有值：

```javascript
(function(){
  var empty = [];
  document.querySelectorAll('.questionLi').forEach(function(q,i){
    var qid = q.getAttribute('data');
    var val = $('#answer'+qid).val();
    if(!val) empty.push(i+1);
  });
  return empty.length ? '未作答: '+empty.join(',') : '全部已作答';
})()
```

## 已知限制

1. **不支持的题型**：填空题、程序题、简答题需单独处理逻辑
2. **预览模式**：某些预览模式下 `submitForm` 可能因 `teacherBrowseMode=true` 被拦截
3. **防作弊检测**：部分考试有切屏检测、人脸监控，自动化操作可能触发风控
4. **逐题模式翻页**：需额外处理"下一题"按钮点击和页面加载等待

## 快速参考

| 操作 | 函数 | 参数 |
|------|------|------|
| 设置单选 | `addChoice($span)` | jQuery span 元素 |
| 设置多选 | `clickSaveMultiSelect(div, qid)` | DOM div 元素 + 题目ID字符串 |
| 提交保存 | `submitForm(true, null, function(){})` | tempSave, quietSubmit, callback |
| 清除选择 | `$('.choice'+qid).removeClass('check_answer'); $('#answer'+qid).val('');` | 题目ID |
| 读取答案 | `$('#answer'+qid).val()` | 题目ID |
