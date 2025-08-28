<%*
/**
 * ACTIVITY TEMPLATE - TEMPLATER VERSION
 * 
 * Why use Templater instead of DataviewJS blocks?
 * ===============================================
 * 1. TEMPLATE INSTANTIATION: Templater executes during file creation, allowing
 *    for immediate setup of activity-specific frontmatter and structure.
 * 2. DYNAMIC CONTENT: Can generate different processing pipelines based on
 *    activity type, stage, or other conditions at creation time.
 * 3. INITIALIZATION CONTROL: Ensures proper activity setup before any user
 *    interaction, establishing consistent activity lifecycle management.
 * 4. PERFORMANCE: One-time execution during creation vs repeated DataviewJS
 *    execution on every file open, improving long-term performance.
 */

// =============================================================================
// ACTIVITY PROCESSING PIPELINE
// =============================================================================

tR += generateActivityProcessingBlock();

// =============================================================================
// HELPER FUNCTION: ACTIVITY CONTENT GENERATION
// =============================================================================

function generateActivityProcessingBlock() {
  let block = "```dataviewjs\n";
  
  // --- Setup and Initialization ---
  block += "// Activity Processing Pipeline\n";
  block += "const {fileIO} = await cJS();\n";
  block += "const currentPageFile = dv.current().file;\n";
  block += "let currentPageContent = await fileIO.loadFile(app, currentPageFile.path);\n\n";

  // --- Frontmatter Configuration ---
  block += "// Initialize activity frontmatter values\n";
  block += "const startDateRaw = dv.current().startDate;\n";
  block += "let startDate = startDateRaw?.toString().format(\"YYYY-MM-DD\");\n";
  block += "if (!startDate) startDate = fileIO.todayDate();\n\n";

  block += "let responsible = dv.current().responsible?.toString();\n";
  block += "if (!responsible) responsible = \"Me\";\n";
  block += "let currentStage = dv.current().stage || \"active\";\n";
  block += "let currentType = dv.current().type || null; // Preserve type field\n\n";

  // --- Content Structure Extraction ---
  block += "// Generate and extract content structure\n";
  block += "let frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible, currentType);\n";
  block += "currentPageContent = currentPageContent.replace(frontmatter, '').trim();\n\n";

  block += "let dataviewJsBlock = \"\";\n";
  block += "if (currentPageContent.trim().length > 0) {\n";
  block += "  ({dataviewJsBlock} = fileIO.extractFrontmatterAndDataviewJs(currentPageContent));\n";
  block += "}\n\n";

  // --- Journal Blocks Parsing ---
  block += "// Parse all journal pages for cross-references and mentions\n";
  block += "const journalPages = dv.pages('\"Journal\"');\n";
  block += "const {noteBlocksParser} = await cJS();\n";
  block += "const allBlocks = await noteBlocksParser.run(app, journalPages, \"YYYY-MM-DD\");\n\n";

  // --- Content Extraction for Processing ---
  block += "// Extract content after dataviewjs block for attribute processing\n";
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

  // --- Mentions and Cross-References Processing (FIRST) ---
  block += "// Process mentions and cross-references from other notes\n";
  block += "const {mentionsProcessor} = await cJS();\n";
  block += "const tagId = currentPageFile.name;\n";
  block += "const frontmatterObj = {\n";
  block += "  startDate: startDate,\n";
  block += "  stage: currentStage,\n";
  block += "  type: currentType, // Include type in frontmatter object\n";
  block += "  responsible: responsible\n";
  block += "};\n";
  block += "const mentions = await mentionsProcessor.run(contentAfterDataview, allBlocks, tagId, frontmatterObj);\n\n";

  block += "// Apply mentions if found\n";
  block += "if (mentions && mentions.trim().length > 0) {\n";
  block += "  contentAfterDataview = mentions;\n";
  block += "}\n\n";

  // --- Attributes Processing (SECOND) ---
  block += "// Process activity attributes and directives (including those from mentions)\n";
  block += "const {attributesProcessor} = await cJS();\n";
  block += "const processedContent = await attributesProcessor.processAttributes(frontmatterObj, contentAfterDataview);\n";
  block += "currentStage = frontmatterObj.stage;\n";
  block += "startDate = frontmatterObj.startDate;\n";
  block += "currentType = frontmatterObj.type; // Update type from processed attributes\n\n";

  block += "// Regenerate frontmatter with processed attributes\n";
  block += "frontmatter = fileIO.generateActivityHeader(startDate, currentStage, responsible, currentType);\n";
  block += "contentAfterDataview = processedContent;\n\n";

  // --- Final Assembly and Save ---
  block += "// Combine all processed elements and save\n";
  block += "const combinedContent = [frontmatter, dataviewJsBlock, contentAfterDataview].join(\"\\n\\n\");\n";
  block += "await fileIO.saveFile(app, currentPageFile.path, combinedContent);\n";
  block += "```\n";

  return block;
}
%>
