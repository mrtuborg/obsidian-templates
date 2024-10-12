---
date_added: <% tp.date.now("YYYY-MM-DD") %>
stage: planning
action: define
timeSlot: on-the-fly
isImportant: true
isUrgent: true
checkpoint: <% tp.date.now("YYYY-MM-DD") %>
deadline: <% tp.date.now("YYYY-MM-DD") %>
responsible:
accountable: 
---
 <% await tp.file.move ("/Activities/Backlog/" + tp.file.title) %>
### Notes
```dataviewjs

```
---
