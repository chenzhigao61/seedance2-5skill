第1步：上传 SKILL.md

点页面上的 "uploading an existing file" 链接
拖入文件或手动创建：文件名填 SKILL.md，内容复制下面：

yaml
1
2
3
4
5
6
7
8
9
10
11
12
13
14
15
16
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
---
name: seedance2-5skill
version: "1.0.0"
description: AI视频与图片一键生成，支持Seedance 2.0/2.0Fast/1.5 Pro文生视频、图生视频，多模型文生图，积分查询与作品管理。当用户提到生成视频、制作视频、AI视频、文生视频、图生视频、生成图片、AI绘图、文生图、查积分、查余额、查看作品、Seedance视频、seedance等需求时使用此技能。
metadata:
  openclaw:
    emoji: "🎬"
    requires:
      bins:
        - python3
      env:
        - AI_DIRECTOR_API_KEY
    homepage: https://aidaoyan88.coze.site
---
# Seedance2.5 Skill - AI视频图片生成
通过AI导演平台MCP API，一键生成AI视频和图片。支持Seedance 2.0/2.0Fast/1.5 Pro视频生成、多模型图片生成。
> **前置准备**：使用本技能前，需先访问 [AI导演平台](https://seedance25move.coze.site) 注册账号 → 获取API Key（sk-开头）→ 充值积分。API Key配置方式因平台而异，见下方说明。
## API Key 配置
| 平台 | 配置方式 |
|------|---------|
| **OpenClaw/WorkBuddy** | 在mcp.json或环境变量中设置 `AI_DIRECTOR_API_KEY=sk-xxx` |
| **扣子Coze** | 安装技能后按提示填写API Key凭证 |
| **ClawHub** | 安装后配置环境变量 `AI_DIRECTOR_API_KEY` |
## 何时使用
- 用户说"生成视频""做个视频""AI视频""文生视频""图生视频""Seedance"
- 用户说"生成图片""AI绘图""文生图"
- 用户说"查积分""查余额""我的作品"
- 用户提供了图片URL并要求基于图片生成视频
## 何时不用
- 用户只是讨论视频/图片编辑（如剪辑、加字幕）
- 用户要求生成GIF或表情包
- 用户要求处理本地视频文件（本技能只做AI生成）
## 工作流程
### 1. 视频生成
```bash
python3 {baseDir}/scripts/generate.py video "提示词" [选项]

表格
参数	说明	默认值
prompt	视频提示词（必填）	-
--model	模型：seedance-2.0, seedance-2.0-fast, seedance-1.5-pro	seedance-2.0
--ratio	比例：16:9, 9:16, 1:1	16:9
--resolution	分辨率：480p, 720p, 1080p	480p
--duration	时长(秒)：4, 6, 8, 10	4
--image-url	参考图片URL（图生视频）	无

文生视频示例：

bash
1
2
python3 {baseDir}/scripts/generate.py video "一只金色凤凰在云海中展翅翱翔，电影级画面，慢动作"

图生视频示例：

bash
1
2
python3 {baseDir}/scripts/generate.py video "镜头缓缓推进，人物微笑转身" --image-url "https://example.com/photo.jpg" --model seedance-2.0

2. 图片生成

bash
1
2
python3 {baseDir}/scripts/generate.py image "提示词" [选项]

表格
参数	说明	默认值
prompt	图片提示词（必填）	-
--model	模型：seedream-3.0, gpt-image-2, flux-kontext-pro等	seedream-3.0
--size	尺寸：1024x1024, 1536x1024, 1024x1536等	1024x1024

3. 查询积分

bash
1
2
python3 {baseDir}/scripts/query.py credits

4. 查询任务状态

bash
1
2
python3 {baseDir}/scripts/query.py status TASK_ID

视频生成通常1-5分钟，每30秒轮询一次，直到状态变为completed或failed。

5. 查看作品列表

bash
1
2
python3 {baseDir}/scripts/query.py works [--type video|image|all] [--limit 10]

重要规则

视频生成是异步任务，提交后必须轮询查询结果
轮询间隔30秒，不要频繁查询
生成消耗积分，余额不足时提醒用户充值
每次只提交一个任务，等完成后再提交下一个
图片URL必须可公开访问
生成失败积分自动退还

积分参考

Seedance 2.0视频：约500-2000积分
Seedance 1.5 Pro视频：约300-1200积分
Seedance 2.0 Fast视频：约300-800积分
图片生成：约50-4000积分
1积分≈0.01元，1:1000兑换比例

错误处理

表格
错误信息	原因	处理方式
"请先登录"	API Key无效	引导检查API Key配置
"积分不足"	余额不够	提醒充值
"生成失败"	模型或参数问题	积分自动退，可重试
"参数错误"	模型/分辨率/比例不匹配	检查参数组合

plaintext
17
18
19
20
21
22
23
24
25
26
27
28
29
30
31
32
33
34
35
36
37
38
39
40
41
42
43
44
45
46
47
48
49
50
51
52
53
54
55
56
57
58
59
60
61
62
63
64
65
66
import sys
import json
import argparse
try:
    from coze_workload_identity import requests
    IS_COZE = True
except ImportError:
    import requests
    IS_COZE = False
API_BASE = "https://aidaoyan88.coze.site"
SKILL_ID = "7659691849480814655"
def get_api_key():
    key = os.getenv(f"COZE_AI_DIRECTOR_KEY_{SKILL_ID}")
    if key and not key.startswith("COZE_CRED_DUMMY"):
        return key
    key = os.getenv("AI_DIRECTOR_API_KEY")
    if key:
        return key
    print(json.dumps({
        "error": "API Key未配置。请设置环境变量 AI_DIRECTOR_API_KEY=sk-xxx，或在扣子技能中配置API Key凭证",
        "help": "访问 https://aidaoyan88.coze.site 注册获取API Key"
    }, ensure_ascii=False))
    sys.exit(1)
def mcp_call(tool_name, arguments, timeout=60):
    api_key = get_api_key()
    headers = {
        "Content-Type": "application/json",
        "X-API-Key": api_key,
    }
    payload = {
        "jsonrpc": "2.0",
        "method": "tools/call",
        "params": {
            "name": tool_name,
            "arguments": arguments,
        },
        "id": 1,
    }
    try:
        response = requests.post(
            f"{API_BASE}/api/mcp",
            headers=headers,
            json=payload,

点 "Commit changes"

第3步：创建 scripts/query.py

同样 "Add file" → "Create new file"
文件名填：scripts/query.py
内容复制：

python
97
98
99
100
101
102
103
104
105
106
107
108
109
110
111
112
113
114
115
116
117
118
119
120
121
122
123
124
125
126
127
128
129
130
131
132
133
#!/usr/bin/env python3
                "message": "⏳ 正在生成中，请30秒后再次查询"
            }, ensure_ascii=False, indent=2))
        else:
            print(json.dumps(result, ensure_ascii=False, indent=2))
    else:
        print(json.dumps({"error": "解析失败"}, ensure_ascii=False))
def list_works(work_type="all", limit=10):
    result = mcp_call("list_works", {"type": work_type, "limit": limit})
    print(json.dumps(result, ensure_ascii=False, indent=2))
if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="Seedance2.5 Skill - 查询工具")
    subparsers = parser.add_subparsers(dest="command", help="查询类型")
    subparsers.add_parser("credits", help="查询积分余额")
    sp = subparsers.add_parser("status", help="查询任务状态")
    sp.add_argument("task_id", help="任务ID")
    wp = subparsers.add_parser("works", help="查看作品列表")
    wp.add_argument("--type", default="all", help="类型: video, image, all")
    wp.add_argument("--limit", type=int, default=10, help="数量")
    args = parser.parse_args()
    if args.command == "credits":
        check_credits()
    elif args.command == "status":
        check_status(args.task_id)
    elif args.command == "works":
        list_works(args.type, args.limit)
    else:
        parser.print_help()
