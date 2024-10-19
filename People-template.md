---
phone: 
email: 
birthday:
---
<%*
tR += "```dataviewjs\n";
tR += "const {noteBlocksParser} = await cJS();\n";
tR += "const journalPages = dv.pages('\"Journal\"');\n";
tR += "const allBlocks = await noteBlocksParser.run(app, journalPages);\n";
tR += "\nconst {mentionsProcessor} = await cJS();\n";
tR += "const tagId = dv.current().file.name;\n"
tR += "await mentionsProcessor.run(app, dv, allBlocks, tagId);\n"
//tR += "\nconst {attributesProcessor} = await cJS();\n";
//tR += "await attributesProcessor.run(app, dv, dv.current().file);\n";
//tR += "\nconst {timelineView} = await cJS();\n";
//tR += "await timelineView.run(dv, app);\n";
tR += "```\n\n";
tR += "# Notes:\n\n";
%>
---
