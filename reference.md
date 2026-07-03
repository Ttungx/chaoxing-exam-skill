# 超星考试 DOM 参考

## 页面整体结构

```
body
├── .topbar (顶部栏：返回/交卷按钮)
├── .examContent (考试主区域)
│   ├── .questionLi[data="题目ID"] (每题一个)
│   │   ├── h3.mark_name (题号 + 题型标签)
│   │   │   ├── span.colorShallow (题型文字)
│   │   │   └── div (题干文本)
│   │   └── form (答题表单)
│   │       ├── input[type=hidden][name="type{qid}"] (0=单选,1=多选)
│   │       ├── input[type=hidden][name="questionId"]
│   │       ├── input[type=hidden][name="typeName{qid}"]
│   │       ├── input[type=hidden][id="answer{qid}"]
│   │       └── .stem_answer > .clearfix.answerBg (选项)
├── .answerCard (答题卡侧边栏)
└── 弹窗 (交卷确认/时间耗尽等)
```

## 关键函数

### addChoice(obj) - 核心选择函数
- 参数: jQuery span 元素
- 行为: toggle check_answer 类, 更新 #answer{qid} 值
- 单选: 先清除同题所有选项再添加
- 多选: 不先清除, 允许叠加

### saveSingleSelect(obj, dataId)
- 调用 addChoice + submitForm(true, div, callBack)
- 注意: 异步 AJAX 可能失败, 建议直接用 addChoice

### clickSaveMultiSelect(obj, dataId)
- 类似 saveSingleSelect, 多选模式

### submitForm(tempSave, quietSubmit, callBack)
- tempSave=true 为暂存, false 为正式提交
- teacherBrowseMode=true 时直接 return, 不保存

## 显示字母 vs 内部值

每道题的 display/data 映射随机, 如:
- Display A -> data="B"
- Display B -> data="A"
必须按显示字母定位选项, 不能假设对应关系。

## 构建题目映射脚本

```javascript
(function(){
    return JSON.stringify(
        Array.from(document.querySelectorAll('.questionLi')).map((q,i) => ({
            idx: i+1,
            id: q.getAttribute('data'),
            type: q.querySelector('input[name^=typeName]')?.value,
            opts: Array.from(q.querySelectorAll('.answerBg')).map(d => ({
                letter: d.querySelector('.num_option, .num_option_dx')?.textContent.trim(),
                data: d.querySelector('.num_option, .num_option_dx')?.getAttribute('data'),
                text: d.querySelector('.answer_p')?.textContent.trim().substring(0,30)
            }))
        }))
    );
})()
```

## 其他题型

- **填空题**: textarea 直接设 value
- **程序题**: 用 codeEditors[questionId].setValue(code)
- **判断题**: 结构同单选, 只有2个选项

## 常见陷阱

1. IIFE 必需: javascript_tool 多语句时必须用 (function(){ return result; })() 包裹
2. AJAX 竞态: 快速连续调用 submitForm 可能冲突, 建议批量后统一提交
3. 视觉状态丢失: submitForm 回调可能重置 DOM, 需恢复 check_answer 类
4. 选项数量: 单选通常4个(ABCD), 多选通常5个(ABCDE)
5. get_page_text 截断: 设置 max_chars: 200000
