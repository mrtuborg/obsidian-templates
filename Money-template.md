
```dataviewjs
const {noteBlocksParser} = await cJS();
const journalPages = dv.pages('"Journal"');
const allBlocks = await noteBlocksParser.run(app, journalPages);

const {mentionsProcessor} = await cJS();
const tagId = dv.current().file.name;
await mentionsProcessor.run(app, dv, allBlocks, tagId);
const {attributesProcessor} = await cJS();
await attributesProcessor.run(app, dv, dv.current().file);
```

---
