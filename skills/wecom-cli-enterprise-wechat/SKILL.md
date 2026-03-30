```markdown
---
name: wecom-cli-enterprise-wechat
description: CLI tool for operating WeCom (企业微信) from the terminal, designed for both humans and AI agents
triggers:
  - use wecom cli to send a message
  - integrate enterprise wechat with my agent
  - manage wecom todos from command line
  - create a wecom meeting via CLI
  - query wecom contacts in terminal
  - wecom-cli setup and configuration
  - operate enterprise wechat schedule with CLI
  - wecom smartsheet management from terminal
---

# wecom-cli Enterprise WeChat CLI

> Skill by [ara.so](https://ara.so) — Daily 2026 Skills collection.

`wecom-cli` is a command-line tool for operating WeCom (企业微信/Enterprise WeChat) from the terminal. It covers contacts, todos, meetings, messages, schedules, documents, and smart sheets — with 12 pre-built AI Agent Skills for direct integration into AI coding agents.

## Installation

```bash
# Install CLI globally
npm install -g @wecom/cli

# Install Agent Skills (required for AI agent use)
npx skills add WeComTeam/wecom-cli -y -g
```

**Requirements:** Node.js with `npm`/`npx`, a WeCom Bot ID and Secret.

## Initial Configuration

Run once to store encrypted credentials locally at `~/.config/wecom/bot.enc`:

```bash
wecom-cli init
# Interactive prompts for Bot ID and Secret

# Or pass bot-id directly
wecom-cli init --bot-id YOUR_BOT_ID
```

## Core CLI Pattern

```bash
# List all categories
wecom-cli --help

# List tools in a category
wecom-cli <category>

# Call a specific tool
wecom-cli <category> <method> '<json_args>'
wecom-cli <category> <method>          # no args variant
```

## Commands by Category

### contact — Contacts (通讯录)

```bash
# Get all visible contacts
wecom-cli contact get_userlist '{}'
```

### todo — Todos (待办)

```bash
# List todos
wecom-cli todo get_todo_list '{}'

# Create a todo with reminder
wecom-cli todo create_todo '{"content": "完成Q2规划文档", "remind_time": "2026-06-01 09:00:00"}'

# Get todo details by IDs
wecom-cli todo get_todo_detail '{"todo_id_list": ["TODO_ID_1", "TODO_ID_2"]}'

# Update todo status (0 = done)
wecom-cli todo update_todo '{"todo_id": "TODO_ID", "todo_status": 0}'

# Delete a todo
wecom-cli todo delete_todo '{"todo_id": "TODO_ID"}'

# Change user status on a todo
wecom-cli todo change_todo_user_status '{"todo_id": "TODO_ID", "status": 1}'
```

### meeting — Meetings (会议)

```bash
# List meetings in a date range (±30 days from today)
wecom-cli meeting list_user_meetings '{
  "begin_datetime": "2026-03-23 00:00",
  "end_datetime": "2026-03-29 23:59",
  "limit": 100
}'

# Create a meeting with invitees
wecom-cli meeting create_meeting '{
  "title": "技术方案评审",
  "meeting_start_datetime": "2026-03-30 15:00",
  "meeting_duration": 3600,
  "invitees": {"userid": ["zhangsan", "lisi"]}
}'

# Get meeting details
wecom-cli meeting get_meeting_info '{"meetingid": "MEETING_ID"}'

# Cancel a meeting
wecom-cli meeting cancel_meeting '{"meetingid": "MEETING_ID"}'

# Update invitees (full replacement)
wecom-cli meeting set_invite_meeting_members '{
  "meetingid": "MEETING_ID",
  "invitees": {"userid": ["zhangsan", "wangwu"]}
}'
```

### msg — Messages (消息)

```bash
# Get chat list for a date range
wecom-cli msg get_msg_chat_list '{
  "begin_time": "2026-03-22 00:00:00",
  "end_time": "2026-03-29 23:59:59"
}'

# Pull message history (chat_type: 1=single, 2=group)
wecom-cli msg get_message '{
  "chat_type": 1,
  "chatid": "zhangsan",
  "begin_time": "2026-03-29 09:00:00",
  "end_time": "2026-03-29 18:00:00"
}'

# Send a text message
wecom-cli msg send_message '{
  "chat_type": 1,
  "chatid": "zhangsan",
  "msgtype": "text",
  "text": {"content": "hello from wecom-cli"}
}'

# Download media file
wecom-cli msg get_msg_media '{"media_id": "MEDIA_ID"}'
```

### schedule — Schedules (日程)

```bash
# Get schedule IDs in range (±30 days)
wecom-cli schedule get_schedule_list_by_range '{
  "start_time": "2026-03-29 00:00:00",
  "end_time": "2026-03-29 23:59:59"
}'

# Get schedule details (1–50 IDs)
wecom-cli schedule get_schedule_detail '{"schedule_id_list": ["SCHEDULE_ID"]}'

# Create a schedule with reminder and attendees
wecom-cli schedule create_schedule '{
  "schedule": {
    "start_time": "2026-03-30 14:00:00",
    "end_time": "2026-03-30 15:00:00",
    "summary": "需求评审",
    "attendees": [{"userid": "zhangsan"}],
    "reminders": {
      "is_remind": 1,
      "remind_before_event_secs": 900,
      "timezone": 8
    }
  }
}'

# Update a schedule (only changed fields)
wecom-cli schedule update_schedule '{
  "schedule_id": "SCHEDULE_ID",
  "schedule": {"summary": "新标题"}
}'

# Cancel a schedule
wecom-cli schedule cancel_schedule '{"schedule_id": "SCHEDULE_ID"}'

# Add attendees
wecom-cli schedule add_schedule_attendees '{
  "schedule_id": "SCHEDULE_ID",
  "attendees": [{"userid": "lisi"}]
}'

# Remove attendees
wecom-cli schedule del_schedule_attendees '{
  "schedule_id": "SCHEDULE_ID",
  "attendees": [{"userid": "lisi"}]
}'

# Check availability for up to 10 users
wecom-cli schedule check_availability '{
  "check_user_list": ["zhangsan", "lisi"],
  "start_time": "2026-03-30 09:00:00",
  "end_time": "2026-03-30 18:00:00"
}'
```

### doc — Documents (文档)

```bash
# Create a document (doc_type=3)
wecom-cli doc create_doc '{"doc_type": 3, "doc_name": "项目周报"}'

# Read document content (async — returns task_id on first call)
wecom-cli doc get_doc_content '{"docid": "DOC_ID", "type": 2}'

# Poll with task_id until ready
wecom-cli doc get_doc_content '{"docid": "DOC_ID", "type": 2, "task_id": "TASK_ID"}'

# Edit document content with Markdown (overwrites body)
wecom-cli doc edit_doc_content '{
  "docid": "DOC_ID",
  "content": "# 标题\n\n正文内容",
  "content_type": 1
}'
```

### doc — Smart Sheets (智能表格)

```bash
# Create a smart sheet (doc_type=10)
wecom-cli doc create_doc '{"doc_type": 10, "doc_name": "任务跟踪表"}'

# List all sub-sheets
wecom-cli doc smartsheet_get_sheet '{"docid": "DOC_ID"}'

# Add a sub-sheet
wecom-cli doc smartsheet_add_sheet '{"docid": "DOC_ID", "title": "Sprint 1"}'

# Rename a sub-sheet
wecom-cli doc smartsheet_update_sheet '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "title": "Sprint 2"
}'

# Delete a sub-sheet (irreversible)
wecom-cli doc smartsheet_delete_sheet '{"docid": "DOC_ID", "sheet_id": "SHEET_ID"}'

# Get fields/columns of a sub-sheet
wecom-cli doc smartsheet_get_fields '{"docid": "DOC_ID", "sheet_id": "SHEET_ID"}'

# Add a field/column
wecom-cli doc smartsheet_add_fields '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "fields": [
    {"field_title": "状态", "field_type": "FIELD_TYPE_SINGLE_SELECT"},
    {"field_title": "负责人", "field_type": "FIELD_TYPE_TEXT"}
  ]
}'

# Update field title
wecom-cli doc smartsheet_update_fields '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "fields": [{"field_id": "FIELD_ID", "field_title": "新标题"}]
}'

# Delete fields (irreversible)
wecom-cli doc smartsheet_delete_fields '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "field_ids": ["FIELD_ID"]
}'

# Query all records
wecom-cli doc smartsheet_get_records '{"docid": "DOC_ID", "sheet_id": "SHEET_ID"}'

# Add records
wecom-cli doc smartsheet_add_records '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "records": [
    {"values": {"标题": [{"type": "text", "text": "新任务"}]}}
  ]
}'

# Update records
wecom-cli doc smartsheet_update_records '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "records": [
    {"record_id": "RECORD_ID", "values": {"状态": [{"type": "text", "text": "完成"}]}}
  ]
}'

# Delete records (irreversible)
wecom-cli doc smartsheet_delete_records '{
  "docid": "DOC_ID",
  "sheet_id": "SHEET_ID",
  "record_ids": ["RECORD_ID"]
}'
```

## AI Agent Skills Reference

These 12 pre-built skills are installed via `npx skills add WeComTeam/wecom-cli -y -g`:

| Skill Name | Category | Purpose |
|---|---|---|
| `wecomcli-lookup-contact` | contact | Search contacts by name/alias |
| `wecomcli-get-todo-list` | todo | List todos with time filter + pagination |
| `wecomcli-get-todo-detail` | todo | Batch fetch todo details |
| `wecomcli-edit-todo` | todo | Create/update/delete/status-change todos |
| `wecomcli-create-meeting` | meeting | Create scheduled meetings |
| `wecomcli-edit-meeting` | meeting | Cancel meetings, update invitees |
| `wecomcli-get-meeting` | meeting | List and get meeting details |
| `wecomcli-get-msg` | msg | Chat list, history, media download, send text |
| `wecomcli-manage-schedule` | schedule | Full schedule CRUD + availability check |
| `wecomcli-manage-doc` | doc | Document create/read/edit |
| `wecomcli-manage-smartsheet-schema` | smartsheet | Sub-sheet and field management |
| `wecomcli-manage-smartsheet-data` | smartsheet | Record CRUD |

## Common Patterns

### Pattern: Poll for async document content

```bash
# Step 1: initiate
RESULT=$(wecom-cli doc get_doc_content '{"docid": "DOC_ID", "type": 2}')
TASK_ID=$(echo $RESULT | jq -r '.task_id')

# Step 2: poll until complete
wecom-cli doc get_doc_content "{\"docid\": \"DOC_ID\", \"type\": 2, \"task_id\": \"$TASK_ID\"}"
```

### Pattern: Create a full smart sheet workflow

```bash
# 1. Create the sheet document
wecom-cli doc create_doc '{"doc_type": 10, "doc_name": "项目看板"}'
# → note the returned docid

# 2. Get the default sub-sheet
wecom-cli doc smartsheet_get_sheet '{"docid": "DOC_ID"}'
# → note the sheet_id

# 3. Add columns
wecom-cli doc smartsheet_add_fields '{
  "docid": "DOC_ID", "sheet_id": "SHEET_ID",
  "fields": [
    {"field_title": "任务", "field_type": "FIELD_TYPE_TEXT"},
    {"field_title": "优先级", "field_type": "FIELD_TYPE_SINGLE_SELECT"},
    {"field_title": "截止日期", "field_type": "FIELD_TYPE_DATE_TIME"}
  ]
}'

# 4. Insert rows
wecom-cli doc smartsheet_add_records '{
  "docid": "DOC_ID", "sheet_id": "SHEET_ID",
  "records": [
    {"values": {"任务": [{"type": "text", "text": "上线前测试"}], "优先级": [{"type": "text", "text": "高"}]}}
  ]
}'
```

### Pattern: Find contacts then send message

```bash
# Get user list to find userid
wecom-cli contact get_userlist '{}'

# Send a message using the userid
wecom-cli msg send_message '{
  "chat_type": 1,
  "chatid": "USERID_FROM_CONTACT_LIST",
  "msgtype": "text",
  "text": {"content": "项目更新通知"}
}'
```

## Configuration Details

- **Credentials file:** `~/.config/wecom/bot.enc` (encrypted)
- **Re-configure:** run `wecom-cli init` again to update credentials
- **Per-bot override:** `wecom-cli init --bot-id SPECIFIC_BOT_ID`

## Troubleshooting

| Problem | Solution |
|---|---|
| `command not found: wecom-cli` | Run `npm install -g @wecom/cli` |
| Auth errors on API calls | Re-run `wecom-cli init` to refresh credentials |
| Skills not available to agent | Run `npx skills add WeComTeam/wecom-cli -y -g` |
| `get_doc_content` returns task_id only | It's async — call again with the `task_id` field |
| Date range errors on meeting/schedule | Range is limited to ±30 days from today |
| JSON parse errors | Ensure JSON args use single-outer / double-inner quotes in shell |

## Shell Escaping Tips

```bash
# Use single quotes around JSON in bash
wecom-cli todo create_todo '{"content": "my task"}'

# For dynamic values, use printf or heredoc
CONTENT="完成报告"
wecom-cli todo create_todo "$(printf '{"content": "%s"}' "$CONTENT")"
```
```
