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
  
  // This is a daily note
  tR += "```dataviewjs\n";
  // Helper class to communicate with Vault files
  tR += "const {fileIO} = await cJS();\n";
  tR += "const currentPageFile = dv.current().file;\n";
  tR += "let currentPageContent = await fileIO.loadFile(app, currentPageFile.path);\n";

// define todayDate if it is not defined yet in existing file
  tR += "const startDateRaw = dv.current().startDate;\n";
  tR += "let startDate = startDateRaw?.toString().format(\"YYYY-MM-DD\");\n";
  tR += "if (!startDate) startDate = fileIO.todayDate();\n";

  tR += "let responsible = dv.current().responsible?.toString();\n";
  tR += "if (!responsible) responsible = \"Me\";\n";
  tR += "let currentStage = dv.current().stage || \"active\";\n";

  tR += "let frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible);\n";
  // Remove the generated header from currentPageContent
  tR += "currentPageContent = currentPageContent.replace(frontmatter, '').trim();\n";

  tR += "let dataviewJsBlock = \"\";\n";
  tR += "let pageContent = \"\";\n";

  tR += "if (currentPageContent.trim().length > 0) {\n";
  tR += "  ({_,dataviewJsBlock,_} = fileIO.extractFrontmatterAndDataviewJs(currentPageContent));\n";
  tR += "};\n";

  //// Note Blocks Parser //////////////////////////////////////////
  // Create data of all pages in journal and parse their blocks
  // We need to find all ToDos to rollover them into today
  //-------------------------------------------------------------
  tR += "\n";
  tR += "const {noteBlocksParser} = await cJS();\n";
  tR += "const journalPages = dv.pages('\"Journal\"').filter(\n";
  tR += `  (page) => !page.file.path.trim().includes(\"${title}\")\n`;
  tR += ");\n";

  tR += "const allBlocks = await noteBlocksParser.run(app, journalPages, \"YYYY-MM-DD\");\n";

  //// Attributes Processor //////////////////////////////////////////
  //------------------------------------------------------------------
  tR += "\n";
  tR += "// Get the content after the dataviewjs block to process attributes\n";
  tR += "let contentAfterDataview = \"\";\n";
  tR += "if (currentPageContent.trim().length > 0) {\n";
  tR += "  const lines = currentPageContent.split('\\n');\n";
  tR += "  let inDataviewBlock = false;\n";
  tR += "  let afterDataview = false;\n";
  tR += "  for (let i = 0; i < lines.length; i++) {\n";
  tR += "    if (lines[i].startsWith('```dataviewjs')) {\n";
  tR += "      inDataviewBlock = true;\n";
  tR += "    } else if (lines[i].startsWith('```') && inDataviewBlock) {\n";
  tR += "      inDataviewBlock = false;\n";
  tR += "      afterDataview = true;\n";
  tR += "    } else if (afterDataview) {\n";
  tR += "      contentAfterDataview += lines[i] + '\\n';\n";
  tR += "    }\n";
  tR += "  }\n";
  tR += "}\n";
  tR += "\n";
  tR += "const {attributesProcessor} = await cJS();\n";
  tR += "const frontmatterObj = {\n";
  tR += "  startDate: startDate,\n";
  tR += "  stage: currentStage,\n";
  tR += "  responsible: responsible\n";
  tR += "};\n";
  tR += "const processedContent = await attributesProcessor.processAttributes(frontmatterObj, contentAfterDataview);\n";
  tR += "// Update values with the processed values\n";
  tR += "currentStage = frontmatterObj.stage;\n";
  tR += "startDate = frontmatterObj.startDate;\n";
  tR += "// Update frontmatter with processed attributes\n";
  tR += "frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible);\n";
  tR += "// Update contentAfterDataview with processed content (directives converted to comments)\n";
  tR += "contentAfterDataview = processedContent;\n";

  //// Mentions processor ///////////////////////////////////////////////////
  // If one day in the past was mentioning today's date
  // - we need to copy all what was assigned to that mention (Reminder)
  //-----------------------------------------------------------------------
  //
  tR += "\nconst {mentionsProcessor} = await cJS();\n";
  tR += "const tagId = currentPageFile.name;\n";

  tR += "const mentions = await mentionsProcessor.run(contentAfterDataview, allBlocks, tagId, frontmatterObj);\n";
  tR += "if (mentions && mentions.trim().length > 0) contentAfterDataview = mentions;\n";
  tR += "// Update frontmatter again after mentions processing (in case directives from other files changed it)\n";
  tR += "frontmatter = fileIO.generateActivityHeader(frontmatterObj.startDate, frontmatterObj.stage, frontmatterObj.responsible);\n";

  /// === Here we need to save currentPageContent to the current file
  tR += "const combinedContent = [frontmatter, dataviewJsBlock, contentAfterDataview].join(\"\\n\\n\");\n";
  tR += "await fileIO.saveFile(app, currentPageFile.path, combinedContent);\n";
  tR += "```\n";
  
} else {
  // This is a daily note
  tR += "```dataviewjs\n";
  // Helper class to communicate with Vault files
  tR += "const {fileIO} = await cJS();\n";
  tR += "const currentPageFile = dv.current().file;\n";
  tR += "let currentPageContent = await fileIO.loadFile(app, currentPageFile.path);\n";

  tR += `let frontmatter = fileIO.generateDailyNoteHeader(\"${title}\");\n`;
  // Remove the generated header from currentPageContent
  tR += "currentPageContent = currentPageContent.replace(frontmatter, '').trim();\n";

  tR += "let dataviewJsBlock = \"\";\n";
  tR += "let pageContent = \"\";\n";

  tR += "if (currentPageContent.trim().length > 0) {\n";
  tR += "  ({_,dataviewJsBlock,pageContent} = fileIO.extractFrontmatterAndDataviewJs(currentPageContent));\n";
  tR += "};\n";

  tR += "const pageIsToday = (fileIO.isDailyNote(currentPageFile.name));\n";
  tR += "const dailyNoteDate = moment(dv.current().name).format(\"YYYY-MM-DD\");\n";

  // Create data of all pages in journal and parse their blocks
  // We need to find all ToDos to rollover them into today
  //-------------------------------------------------------------
  tR += "\n";
  tR += "const {noteBlocksParser} = await cJS();\n";
  tR += "const journalPages = dv.pages('\"Journal\"').filter(\n";
  tR += `  (page) => !page.file.path.trim().includes(\"${title}\")\n`;
  tR += ");\n";

  tR += "const allBlocks = await noteBlocksParser.run(app, journalPages, \"YYYY-MM-DD\");\n";

  // Todo Rollover 
  //-------------------------------------------------------------
  tR += "\n";
  tR += "if (pageIsToday) {\n";
  tR += "  const {todoRollover} = await cJS();\n";
  tR += "  const remove = true;\n";
  tR += "  const todos = await todoRollover.run(app, allBlocks, dailyNoteDate, pageContent, remove);\n";
  tR += "  pageContent += \"\" + todos;\n";
  tR += "}\n";

  // Add activities under focus
  // TODO: Refresh activities pages before to update all attributes
  //-------------------------------------------------------------
  tR += "\n";
  tR += "if (pageIsToday) {\n";
  tR += "  const {activitiesInProgress} = await cJS();\n";
  tR += "  const activities = await activitiesInProgress.run(app, pageContent);\n";
  tR += "if (activities && activities.trim().length > 0) pageContent = activities;\n";
  tR += "}\n";

  // Mentions processor. If one day in the past was mentioning today's date
  // - we need to copy all what was assigned to that mention (Reminder)
  //-----------------------------------------------------------------------
  // Not sure if I need to check if this is today or not. Decide later.
  //
  tR += "\nconst {mentionsProcessor} = await cJS();\n";
  tR += "const tagId = currentPageFile.name;\n";

  tR += "const mentions = await mentionsProcessor.run(pageContent, allBlocks, tagId);\n";
  tR += "if (mentions && mentions.trim().length > 0) pageContent = mentions;\n";

  // We need to prepare daily not only once - at creation
  // So, now we need to remove all the scripts
  //-------------------------------------------------------
  tR += "\n";
  tR += "if (pageIsToday) {\n";
  tR += "  const {scriptsRemove} = await cJS();\n";
  tR += "  dataviewJsBlock = await scriptsRemove.run(dataviewJsBlock);\n";
  tR += "}\n";

  /// === Here we need to save currentPageContent to the current file
  tR += "const combinedContent = [frontmatter, dataviewJsBlock];\n";
  tR += "if (!pageContent.includes('----')) {\n";
  tR += "  combinedContent.push('----');\n";
  tR += "}\n";
  tR += "combinedContent.push(pageContent);\n";
  tR += "const combinedContentStr = combinedContent.join(\"\\n\");\n";
  tR += "await fileIO.saveFile(app, currentPageFile.path, combinedContentStr);\n";
  tR += "```\n";
}
%>
