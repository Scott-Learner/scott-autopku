---
name: autopku-tool-data-parser
description: 解析 pku3b 输出数据（ANSI颜色码、作业、公告、课程内容和课程筛选）
---

# 教学网数据解析

## ANSI 颜色码清理

```python
import re

def strip_ansi(text):
    """移除 ANSI 颜色码"""
    return re.sub(r'\x1b\[[0-9;]*m', '', text)
```

## 作业列表解析

```python
import re
import json

def parse_assignments(raw_text):
    """
    解析 pku3b a ls -a 输出
    
    Returns:
        [{"course": "课程名", "assignment": "作业名", "status": "状态", "id": "作业ID"}]
    """
    text = strip_ansi(raw_text)
    pattern = re.compile(
        r'^(?P<course>.+?)\s+>\s+(?P<assignment>.+?)\s+'
        r'\((?P<status>.*)\)\s+(?P<id>[0-9a-f]{1,16})\s*$',
        re.MULTILINE,
    )
    
    return [
        {key: value.strip() for key, value in match.groupdict().items()}
        for match in pattern.finditer(text)
    ]
```

## 选课列表解析

```python
def parse_courses(raw_text):
    """
    解析 pku3b s -d major show 输出
    提取"已选上"的课程
    """
    courses = []
    for line in strip_ansi(raw_text).splitlines():
        if not line.startswith('已选上'):
            continue
        fields = re.split(r'\s{2,}', line[len('已选上'):].strip())
        if fields and fields[0]:
            courses.append(fields[0])
    return courses
```

## 公告解析

```python
def parse_announcements(raw_text):
    """
    解析 pku3b ann ls 输出
    
    Returns:
        [{"course": "课程名", "title": "公告标题", "id": "公告ID"}]
    """
    text = strip_ansi(raw_text)
    pattern = re.compile(
        r'^\[\s*\d+\]\s+(?P<course>.+?)\s+>\s+'
        r'(?P<title>.*?)\s+(?P<id>[0-9a-f]{1,16})\s*'
        r'(?=^\[\s*\d+\]|\Z)',
        re.MULTILINE | re.DOTALL,
    )
    
    return [
        {
            "course": match.group('course').strip(),
            "title": ' '.join(match.group('title').split()),
            "id": match.group('id'),
        }
        for match in pattern.finditer(text)
    ]
```

## 课程内容解析

`pku3b cc ls` 先输出带学期后缀的课程标题，再输出该课程的内容条目。解析时去掉最后一个半角括号学期后缀，使课程名与 `a ls`、`ann ls` 对齐；每条内容以复合 ID `course_id:content_id` 结尾。

```python
def parse_course_contents(raw_text):
    """
    解析 pku3b cc ls 输出。

    Returns:
        [{"course": "课程名", "kind": "File", "title": "课件.pdf",
          "attachment_count": 0, "ccid": "course_id:content_id"}]
    """
    text = strip_ansi(raw_text)
    current_course = None
    contents = []
    item_pattern = re.compile(
        r'^•\s+\((?P<kind>[^)]+)\)\s+(?P<title>.*?)'
        r'(?:\s+\[(?P<attachments>\d+)\s+附件\])?\s+'
        r'(?P<ccid>\S+:\S+)\s*$'
    )

    for raw_line in text.splitlines():
        line = raw_line.strip()
        if not line:
            continue
        match = item_pattern.match(line)
        if match:
            if current_course is None:
                raise ValueError('course content item appeared before course title')
            item = match.groupdict()
            contents.append({
                "course": current_course,
                "kind": item["kind"].strip(),
                "title": item["title"].strip(),
                "attachment_count": int(item["attachments"] or 0),
                "ccid": item["ccid"],
            })
        else:
            current_course = line.rsplit('(', 1)[0].strip()

    return contents
```

## 课程筛选

```python
def filter_current_semester(assignments, current_courses):
    """
    筛选当前学期作业
    同时检查"未选上"但有作业的课程
    """
    current = [a for a in assignments if a['course'] in current_courses]
    
    # 检查遗漏课程
    all_courses = set(a['course'] for a in assignments)
    missed = all_courses - set(current_courses)
    
    return current, list(missed)
```

## 附件检测

```python
def has_attachment(raw_text, course_name):
    """检查课程是否有可下载附件"""
    text = strip_ansi(raw_text)
    pattern = rf'{re.escape(course_name)}.*?\[附件\]'
    return bool(re.search(pattern, text, re.DOTALL))
```
