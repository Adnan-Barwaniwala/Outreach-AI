# Stage 1 Config

## Tally Form

| Item | Value |
|---|---|
| Form Name | OltaFlock - Lead Requests |
| Form ID | gDZAEN |
| Public Form URL | https://tally.so/r/gDZAEN |

## Field Reference

| Field | Tally Key | Array Index |
|---|---|---|
| businessType | question_QrDzL7 | fields[0] |
| location | question_91WRVQ | fields[1] |
| leadCount | question_eA604e | fields[2] |

## n8n Workflow

| Item | Value |
|---|---|
| Instance | adnan-barwaniwala.app.n8n.cloud |
| Workflow Name | oltaflock-pipeline |
| Production Webhook URL | https://adnan-barwaniwala.app.n8n.cloud/webhook/stage1-submit |

## Test Log

| Date | Business Type | Location | Lead Count | Result |
|---|---|---|---|---|
| 2026-04-01 | Groceries | India | 100 | Pass — routed to True branch |
