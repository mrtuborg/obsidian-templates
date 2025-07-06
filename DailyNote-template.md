```dataviewjs
const {dailyNoteComposer} = await cJS();
const currentPageFile = dv.current().file;
const title = currentPageFile.name;
await dailyNoteComposer.processDailyNote(app, dv, currentPageFile, title);
```
