<%*
tR += "\n";
tR += "```dataviewjs\n";
tR += "const {mentionsProcessor} = await cJS();\n";
tR += "const shortTagId = dv.current().file.name;\n";
tR += "const longTagId = dv.current().file.folder + '/' + dv.current().file.name;\n";
tR += "await mentionsProcessor.run(dv, app, dv.pages('\"Journal\"'), shortTagId);\n";
tR += "await mentionsProcessor.run(dv, app, dv.pages('\"Journal\"'), longTagId);\n";
tR += "\n";
tR += "const {attributesProcessor} = await cJS();\n";
tR += "await attributesProcessor.run(app, dv, dv.current().file);\n";
tR += "```\n";
%>
---
