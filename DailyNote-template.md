<%*
/**
 * DAILY NOTE TEMPLATE - TEMPLATER VERSION
 * 
 * Why use Templater instead of DataviewJS blocks?
 * ===============================================
 * 1. FILE MOVEMENT: Templater can move/rename files during template instantiation,
 *    which is impossible with DataviewJS blocks that execute after file creation.
 * 2. CONDITIONAL LOGIC: Template can decide file destination based on filename format
 *    before any content is written, enabling smart file organization.
 * 3. TEMPLATE EXECUTION: Runs once during file creation vs DataviewJS that runs
 *    every time the file is opened, providing better performance for setup logic.
 * 4. CONTENT GENERATION: Can dynamically generate different template content
 *    based on conditions, creating truly adaptive templates.
 */

// =============================================================================
// CONFIGURATION & SETUP
// =============================================================================

const title = tp.file.title;
const activitiesFolder = 'Activities';
const dailyNoteFormat = "YYYY-MM-DD";

// =============================================================================
// MAIN TEMPLATE LOGIC: FILE TYPE DETECTION & ROUTING
// =============================================================================

if (!moment(title, dailyNoteFormat, true).isValid()) {
  // -------------------------------------------------------------------------
  // NON-DAILY NOTE: Move to Activities folder and apply activity processing
  // -------------------------------------------------------------------------
  
  const newPath = `${activitiesFolder}/${title}`;
  
  // Check if file already exists
  const fileExists = await app.vault.adapter.exists(newPath + '.md');

  if (!fileExists) {
    await tp.file.move(newPath);
    console.log(`File moved to: ${newPath}`);
  } else {
    console.log(`File already exists: ${newPath}, move skipped.`);
  }
  
  // Generate Activity frontmatter and processing block
  tR += generateActivityFrontmatter();
  tR += "\n\n";
  tR += generateActivityDataviewBlock(title);

} else {
  // -------------------------------------------------------------------------
  // DAILY NOTE: Apply full daily note processing pipeline
  // -------------------------------------------------------------------------
  
  tR += generateDailyNoteProcessingBlock(title);
}

// =============================================================================
// HELPER FUNCTIONS: CONTENT GENERATION
// =============================================================================

function generateActivityFrontmatter() {
  const today = moment().format("YYYY-MM-DD");
  return `---
startDate: ${today}
stage: active
responsible: [Me]
---`;
}

function generateActivityDataviewBlock(title) {
  return `\`\`\`dataviewjs
const cjsResult = await cJS();
const fileIO = cjsResult.createfileIOInstance();
const currentPageFile = dv.current().file;
let currentPageContent = await fileIO.loadFile(app, currentPageFile.path);
const startDateRaw = dv.current().startDate;
let startDate = startDateRaw?.toString().format("YYYY-MM-DD");
if (!startDate) startDate = fileIO.todayDate();
let responsible = dv.current().responsible?.toString();
if (!responsible) responsible = "Me";
let currentStage = dv.current().stage || "active";
let frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible);
currentPageContent = currentPageContent.replace(frontmatter, '').trim();
let dataviewJsBlock = "";
let pageContent = "";
if (currentPageContent.trim().length > 0) {
  ({dataviewJsBlock} = fileIO.extractFrontmatterAndDataviewJs(currentPageContent));
};

// Get content after dataviewjs block for attribute processing
let contentAfterDataview = "";
if (currentPageContent.trim().length > 0) {
  const lines = currentPageContent.split('\\n');
  let inDataviewBlock = false;
  let afterDataview = false;
  for (let i = 0; i < lines.length; i++) {
    if (lines[i].startsWith('\`\`\`dataviewjs')) {
      inDataviewBlock = true;
    } else if (lines[i].startsWith('\`\`\`') && inDataviewBlock) {
      inDataviewBlock = false;
      afterDataview = true;
    } else if (afterDataview) {
      contentAfterDataview += lines[i] + '\\n';
    }
  }
}

// Process attributes
const attributesProcessor = cjsResult.createattributesProcessorInstance();
const frontmatterObj = {
  startDate: startDate,
  stage: currentStage,
  responsible: responsible
};
const processedContent = await attributesProcessor.processAttributes(frontmatterObj, contentAfterDataview);
currentStage = frontmatterObj.stage;
startDate = frontmatterObj.startDate;
frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible);
contentAfterDataview = processedContent;

const noteBlocksParser = cjsResult.createnoteBlocksParserInstance();
const journalPages = dv.pages('"Journal"').filter(
  (page) => !page.file.path.trim().includes("${title}")
);
const allBlocks = await noteBlocksParser.run(app, journalPages, "YYYY-MM-DD");

const mentionsProcessor = cjsResult.creatementionsProcessorInstance();
const tagId = currentPageFile.name;
const mentions = await mentionsProcessor.run(contentAfterDataview, allBlocks, tagId, frontmatterObj);
if (mentions && mentions.trim().length > 0) contentAfterDataview = mentions;
frontmatter = fileIO.generateActivityHeader(frontmatterObj.startDate, frontmatterObj.stage, frontmatterObj.responsible);
const combinedContent = [frontmatter, dataviewJsBlock, contentAfterDataview].join("\\n\\n");
await fileIO.saveFile(app, currentPageFile.path, combinedContent);
\`\`\``;
}

function generateActivityProcessingBlock(title) {
  let block = "```dataviewjs\n";
  
  // --- Setup and Initialization ---
  block += "// Activity Processing Pipeline\n";
  block += "const {fileIO} = await cJS();\n";
  block += "const currentPageFile = dv.current().file;\n";
  block += "let currentPageContent = await fileIO.loadFile(app, currentPageFile.path);\n\n";

  // --- Frontmatter Configuration ---
  block += "// Initialize frontmatter values\n";
  block += "const startDateRaw = dv.current().startDate;\n";
  block += "let startDate = startDateRaw?.toString().format(\"YYYY-MM-DD\");\n";
  block += "if (!startDate) startDate = fileIO.todayDate();\n\n";

  block += "let responsible = dv.current().responsible?.toString();\n";
  block += "if (!responsible) responsible = \"Me\";\n";
  block += "let currentStage = dv.current().stage || \"active\";\n\n";

  // --- Content Structure Extraction ---
  block += "// Generate and extract content structure\n";
  block += "let frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible);\n";
  block += "currentPageContent = currentPageContent.replace(frontmatter, '').trim();\n\n";

  block += "let dataviewJsBlock = \"\";\n";
  block += "if (currentPageContent.trim().length > 0) {\n";
  block += "  ({dataviewJsBlock} = fileIO.extractFrontmatterAndDataviewJs(currentPageContent));\n";
  block += "}\n\n";

  // --- Journal Blocks Parsing ---
  block += "// Parse all journal pages for cross-references\n";
  block += "const {noteBlocksParser} = await cJS();\n";
  block += "const journalPages = dv.pages('\"Journal\"').filter(\n";
  block += `  (page) => !page.file.path.trim().includes(\"${title}\")\n`;
  block += ");\n";
  block += "const allBlocks = await noteBlocksParser.run(app, journalPages, \"YYYY-MM-DD\");\n\n";

  // --- Attributes Processing ---
  block += "// Extract and process content attributes\n";
  block += "let contentAfterDataview = \"\";\n";
  block += "if (currentPageContent.trim().length > 0) {\n";
  block += "  const lines = currentPageContent.split('\\n');\n";
  block += "  let inDataviewBlock = false;\n";
  block += "  let afterDataview = false;\n";
  block += "  \n";
  block += "  for (let i = 0; i < lines.length; i++) {\n";
  block += "    if (lines[i].startsWith('```dataviewjs')) {\n";
  block += "      inDataviewBlock = true;\n";
  block += "    } else if (lines[i].startsWith('```') && inDataviewBlock) {\n";
  block += "      inDataviewBlock = false;\n";
  block += "      afterDataview = true;\n";
  block += "    } else if (afterDataview) {\n";
  block += "      contentAfterDataview += lines[i] + '\\n';\n";
  block += "    }\n";
  block += "  }\n";
  block += "}\n\n";

  block += "// Process attributes and directives\n";
  block += "const {attributesProcessor} = await cJS();\n";
  block += "const frontmatterObj = {\n";
  block += "  startDate: startDate,\n";
  block += "  stage: currentStage,\n";
  block += "  responsible: responsible\n";
  block += "};\n\n";

  block += "const processedContent = await attributesProcessor.processAttributes(frontmatterObj, contentAfterDataview);\n";
  block += "currentStage = frontmatterObj.stage;\n";
  block += "startDate = frontmatterObj.startDate;\n";
  block += "frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible);\n";
  block += "contentAfterDataview = processedContent;\n\n";

  // --- Mentions Processing ---
  block += "// Process mentions and cross-references\n";
  block += "const {mentionsProcessor} = await cJS();\n";
  block += "const tagId = currentPageFile.name;\n";
  block += "const mentions = await mentionsProcessor.run(contentAfterDataview, allBlocks, tagId, frontmatterObj);\n\n";

  block += "if (mentions && mentions.trim().length > 0) {\n";
  block += "  contentAfterDataview = mentions;\n";
  block += "}\n\n";

  block += "// Update frontmatter after mentions processing\n";
  block += "frontmatter = fileIO.generateActivityHeader(frontmatterObj.startDate, frontmatterObj.stage, frontmatterObj.responsible);\n\n";

  // --- Final Assembly and Save ---
  block += "// Combine and save final content\n";
  block += "const combinedContent = [frontmatter, dataviewJsBlock, contentAfterDataview].join(\"\\n\\n\");\n";
  block += "await fileIO.saveFile(app, currentPageFile.path, combinedContent);\n";
  block += "```\n";

  return block;
}

function generateDailyNoteProcessingBlock(title) {
  let block = "```dataviewjs\n";
  
  // --- Setup and Initialization ---
  block += "// Daily Note Processing Pipeline\n";
  block += "const cjsResult = await cJS();\n";
  block += "const fileIO = cjsResult.createfileIOInstance();\n";
  block += "const currentPageFile = dv.current().file;\n";
  block += "let currentPageContent = await fileIO.loadFile(app, currentPageFile.path);\n\n";

  // --- Frontmatter Generation ---
  block += "// Generate daily note frontmatter\n";
  block += `let frontmatter = fileIO.generateDailyNoteHeader(\"${title}\");\n`;
  block += "currentPageContent = currentPageContent.replace(frontmatter, '').trim();\n\n";

  // --- Content Structure Extraction ---
  block += "let dataviewJsBlock = \"\";\n";
  block += "let pageContent = \"\";\n";
  block += "if (currentPageContent.trim().length > 0) {\n";
  block += "  ({dataviewJsBlock, pageContent} = fileIO.extractFrontmatterAndDataviewJs(currentPageContent));\n";
  block += "}\n\n";

  // --- Date and Context Setup ---
  block += "// Determine if this is today's note\n";
  block += "const pageIsToday = fileIO.isDailyNote(currentPageFile.name);\n";
  block += "const dailyNoteDate = moment(dv.current().name).format(\"YYYY-MM-DD\");\n\n";

  // --- Journal Blocks Parsing ---
  block += "// Parse all journal pages for processing\n";
  block += "const noteBlocksParser = cjsResult.createnoteBlocksParserInstance();\n";
  block += "const journalPages = dv.pages('\"Journal\"').filter(\n";
  block += `  (page) => !page.file.path.trim().includes(\"${title}\")\n`;
  block += ");\n";
  block += "const allBlocks = await noteBlocksParser.run(app, journalPages, \"YYYY-MM-DD\");\n\n";

  // --- Todo Rollover (Today Only) ---
  // block += "// Todo Rollover - Only for today's note\n";
  // block += "if (pageIsToday) {\n";
  // block += "  const {todoRollover} = await cJS();\n";
  // block += "  const remove = true;\n";
  // block += "  \n";
  // block += "  console.log(\"Starting todo rollover for:\", dailyNoteDate);\n";
  // block += "  console.log(\"Total blocks found:\", allBlocks.length);\n";
  // block += "  console.log(\"Todo blocks:\", allBlocks.filter((b) => b.blockType === \"todo\").length);\n";
  // block += "  \n";
  // block += "  const todos = await todoRollover.run(app, allBlocks, dailyNoteDate, pageContent, remove);\n";
  // block += "  console.log(\"Todo rollover result:\", typeof todos, todos ? todos.length : 0);\n";
  // block += "  \n";
  // block += "  if (todos && typeof todos === \"string\" && todos.trim().length > 0) {\n";
  // block += "    pageContent = todos;\n";
  // block += "    console.log(\"Todo rollover completed successfully\");\n";
  // block += "  } else {\n";
  // block += "    console.log(\"No todos to rollover or empty result\");\n";
  // block += "  }\n";
  // block += "}\n\n";

  // --- Activities Integration (Today Only) ---
  block += "// Add activities in progress - Only for today's note\n";
  block += "if (pageIsToday) {\n";
  block += "  // Sync activity todos before copying to daily note\n";
  block += "  const todoSyncManager = cjsResult.createtodoSyncManagerInstance();\n";
  block += "  await todoSyncManager.run(app);\n";
  block += "  \n";
  block += "  const activitiesInProgress = cjsResult.createactivitiesInProgressInstance();\n";
  block += "  const activities = await activitiesInProgress.run(app);\n";
  block += "  \n";
  block += "  if (activities && activities.trim().length > 0) {\n";
  block += "    pageContent = activities;\n";
  block += "  }\n";
  block += "}\n\n";

  // --- Mentions Processing ---
  block += "// Process mentions and reminders\n";
  block += "const mentionsProcessor = cjsResult.creatementionsProcessorInstance();\n";
  block += "const tagId = currentPageFile.name;\n";
  block += "const mentions = await mentionsProcessor.run(pageContent, allBlocks, tagId);\n\n";

  block += "if (mentions && mentions.trim().length > 0) {\n";
  block += "  pageContent = mentions;\n";
  block += "}\n\n";

  // --- Script Cleanup (Today Only) ---
  block += "// Remove processing scripts - Only for today's note\n";
  block += "if (pageIsToday) {\n";
  block += "  const scriptsRemove = cjsResult.createscriptsRemoveInstance();\n";
  block += "  dataviewJsBlock = await scriptsRemove.run(dataviewJsBlock);\n";
  block += "}\n\n";

  // --- Final Assembly and Save ---
  block += "// Combine and save final content\n";
  block += "const combinedContent = [frontmatter, dataviewJsBlock];\n";
  block += "if (!pageContent.includes('----')) {\n";
  block += "  combinedContent.push('----');\n";
  block += "}\n";
  block += "combinedContent.push(pageContent);\n";
  block += "const combinedContentStr = combinedContent.join(\"\\n\");\n";
  block += "await fileIO.saveFile(app, currentPageFile.path, combinedContentStr);\n";
  block += "```\n";

  return block;
}
%>
