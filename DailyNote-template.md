```dataviewjs
const {dailyNoteComposer} = await cJS();
const {activityComposer} = await cJS();
const currentPageFile = dv.current().file;
const title = currentPageFile.name;

// Define the expected date format
const dayFormat = "YYYY-MM-DD";

if (!moment(title, dayFormat, true).isValid()) {
  // This note was created from dailyNote but doesn't belong to journal and should be moved out
  const key = 'Activities';
  const newPath = `${key}/${title}`;
  
  // Check if file already exists
  const fileExists = await app.vault.adapter.exists(newPath + '.md');

  if (!fileExists) {
    // Get current content before moving (preserve any user content)
    const {fileIO} = await cJS();
    const currentContent = await fileIO.loadFile(app, currentPageFile.path);
    
    // Extract any user content (skip the dataviewjs block)
    let userContent = "";
    const lines = currentContent.split('\n');
    let inDataviewBlock = false;
    let afterDataview = false;
    
    for (let i = 0; i < lines.length; i++) {
      if (lines[i].startsWith('```dataviewjs')) {
        inDataviewBlock = true;
      } else if (lines[i].startsWith('```') && inDataviewBlock) {
        inDataviewBlock = false;
        afterDataview = true;
      } else if (afterDataview && lines[i].trim() !== '') {
        userContent += lines[i] + '\n';
      }
    }
    
    // Move the file to Activities folder
    await app.fileManager.renameFile(currentPageFile, newPath + '.md');
    console.log(`File moved to: ${newPath}.md`);
    
    // Load Activity template and replace content
    const activityTemplate = await fileIO.loadFile(app, 'Engine/Templates/Activity-template.md');
    
    // Combine activity template with any user content
    let newContent = activityTemplate;
    if (userContent.trim().length > 0) {
      newContent += '\n\n' + userContent.trim();
    }
    
    // Save the new content with Activity template
    await fileIO.saveFile(app, newPath + '.md', newContent);
    console.log(`Activity template applied to: ${newPath}.md`);
    
    // Get the moved file reference and process it
    const movedFile = app.vault.getAbstractFileByPath(newPath + '.md');
    if (movedFile) {
      // Refresh the file reference for dataview
      await app.workspace.getLeaf().openFile(movedFile);
      console.log(`Activity file opened: ${newPath}.md`);
    }
    
  } else {
    console.log(`File already exists: ${newPath}.md, move skipped.`);
    // Delete the duplicate and open the existing one
    await app.vault.delete(currentPageFile);
    const existingFile = app.vault.getAbstractFileByPath(newPath + '.md');
    if (existingFile) {
      await app.workspace.getLeaf().openFile(existingFile);
    }
  }
  
} else {
  // This is a daily note - use dailyNoteComposer
  await dailyNoteComposer.processDailyNote(app, dv, currentPageFile, title);
}
```
