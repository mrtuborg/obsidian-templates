<%*
let title = tp.file.title;
const key = 'Activities';

// Define the expected date formats
const dayFormat = "YYYY-MM-DD";

if (!moment(title, dayFormat, true).isValid()) {
// This note was created from dailyNote but not belong
// to journal and should be moved out
  const newPath = `${key}/${title}`;
  
  // Check if file already exists
  const fileExists = await app.vault.adapter.exists(newPath);

  if (!fileExists) {
    await tp.file.move(newPath);
    console.log(`File moved to: ${newPath}`);
  } else {
    console.log(`File already exists: ${newPath}, move skipped.`);
  }
  
  // Use centralized activity composer
  tR += "```dataviewjs\n";
  tR += "const {activityComposer} = await cJS();\n";
  tR += "const currentPageFile = dv.current().file;\n";
  tR += "await activityComposer.processActivity(app, dv, currentPageFile);\n";
  tR += "```\n";
  
} else {
  // Use centralized daily note composer
  tR += "```dataviewjs\n";
  tR += "const {dailyNoteComposer} = await cJS();\n";
  tR += "const currentPageFile = dv.current().file;\n";
  tR += `await dailyNoteComposer.processDailyNote(app, dv, currentPageFile, \"${title}\");\n`;
  tR += "```\n";
}
%>
