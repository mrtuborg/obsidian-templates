---
stage: action
---

<%*
tR += "```dataviewjs\n";
tR += "//const {stageProcessor} = await cJS();\n";
tR += "//await stageProcessor.run(app, dv, dv.current().file);\n";
tR += "//\n";
tR += "const {mentionsProcessor} = await cJS();\n";
tR += "const shortTagId = dv.current().file.name;\n";
tR += "await mentionsProcessor.run(dv, app, dv.pages('\"Journal\"'), shortTagId);\n";
tR += "```\n";
%>
