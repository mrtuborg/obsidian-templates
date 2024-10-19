---
stage: action
---

<%*
tR += "```dataviewjs\n";
tR += "const {noteBlocksParser} = await cJS();\n";
tR += "const journalPages = dv.pages('\"Journal\"');\n";
tR += "const allBlocks = await noteBlocksParser.run(app, journalPages);\n";
tR += "\nconst {mentionsProcessor} = await cJS();\n";
tR += "const tagId = dv.current().file.name;\n";
tR += "await mentionsProcessor.run(app, dv, allBlocks, tagId);\n";
tR += "```\n\n";
tR += "# Notes:\n\n";
%>
