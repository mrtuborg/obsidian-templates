```dataviewjs
const {activityComposer} = await cJS();
const currentPageFile = dv.current().file;
await activityComposer.processActivity(app, dv, currentPageFile);
```

