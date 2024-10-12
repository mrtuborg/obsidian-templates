---
phone: 
email: 
birthday:
---

### Notes
<%*
tR += "```dataviewjs\n";
tR += "const {mentionsProcessor} = await cJS();\n";
tR += "const shortTagId = dv.current().file.name;\n";
tR += "await mentionsProcessor.run(dv, app, dv.pages('\"Journal\"'), shortTagId);\n";
tR += "\nconst {attributesProcessor} = await cJS();\n";
tR += "await attributesProcessor.run(app, dv, dv.current().file);\n";
tR += "\nconst {timelineView} = await cJS();\n";
tR += "\nawait timelineView.run(dv, app);\n";
tR += "```\n";
%>
---
