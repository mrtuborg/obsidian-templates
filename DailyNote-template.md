<%*
let title = tp.file.title;
const key = 'Backlog';

// Define the expected date formats
const dayFormat = "YYYY-MM-DD";

if (!moment(title, dayFormat, true).isValid()) {
// This not was created from dailyNote but not belong
// to journal and should be moved out
  const newPath = `${key}/${title}`;
  await tp.file.move(newPath);
  tR += "---\nstage: action\n---\n"
  tR += "```dataviewjs\n";
  tR += "//const {stageProcessor} = await cJS();\n";
  tR += "//await stageProcessor.run(app, dv, dv.current().file);\n";
  tR += "//\n";
  tR += "const {mentionsProcessor} = await cJS();\n";
  tR += "const shortTagId = dv.current().file.name;\n";
  tR += "await mentionsProcessor.run(dv, app, dv.pages('\"Journal\"'), shortTagId);\n";
  tR += "```\n";
  tR += "---\n";
} else {
  const year = moment(tp.file.title,'YYYY-MM-DD').format("YYYY");
  const month = moment(tp.file.title,'YYYY-MM-DD').format("YYYY-MM");
  const monthStr = moment(tp.file.title,'YYYY-MM-DD').format("MMMM");
  const week = moment(tp.file.title,'YYYY-MM-DD').format("YYYY-[W]W");
  const weekNum = moment(tp.file.title,'YYYY-MM-DD').format("WW");
  const dayStr = moment(tp.file.title,'YYYY-MM-DD').format("DD");
  tR += "---\n";
  tR += "---\n";
  tR += `### ${dayStr} [[${month}|${monthStr}]] [[${year}]]\n#### Week: [[${week}|${weekNum}]]\n`;

  tR += "```dataviewjs\n";
  tR += "const { noteBlocksParser } = await cJS();\n";
  tR += "const journalPages = dv.pages('\"Journal\"');\n";
  tR += "const allBlocks = await noteBlocksParser.run(app, journalPages);\n";
  tR += "\nconst {todoRollover} = await cJS();\n";
  tR += "const remove = false;\n";
  tR += "await todoRollover.run(app, dv, allBlocks, remove);\n";
  //tR += "\nconst {mentionsProcessor} = await cJS();\n";
  //tR += "const shortTagId = dv.current().file.name;\n";
  //tR += "await mentionsProcessor.run(dv, app, journalPages, shortTagId);\n";
  //tR += "\nconst {forceStaticPage} = await cJS();\n";
  //tR += "await forceStaticPage.run(dv, app);\n";
  tR += "```\n";
  tR += "\n---\n";
  tR += "# Notes:\n\n";
}
%>

