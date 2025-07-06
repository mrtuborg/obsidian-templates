<%*
// Activity template using centralized activityComposer
tR += "```dataviewjs\n";
tR += "const {activityComposer} = await cJS();\n";
tR += "const currentPageFile = dv.current().file;\n";
tR += "await activityComposer.processActivity(app, dv, currentPageFile);\n";
tR += "```\n";
%>
