---
phone: 
email: 
birthday:
---

## Activities
```dataviewjs
const currentPerson = dv.current().file.name;
const today = moment(new Date());

let defaultCheckpoint = today;
let defaultDeadline = today;

dv.table(["stage", "title", "checkpoint", "deadline"],
	dv.pages('"Activities"')
	.where(b => 
		(b.file.frontmatter) &&
		((b.file.frontmatter.accountable && b.file.frontmatter.accountable == currentPerson) ||
		(b.file.frontmatter.responsible && b.file.frontmatter.responsible == currentPerson))
	)
	.sort(b => b.file.frontmatter.deadline, "asc")
	.map(b=>
		[ b.file.frontmatter.stage,
		 '<nobr>' + b.file.link+'</nobr>',
		(b.file.frontmatter.checkpoint) ?
			moment(b.file.frontmatter.checkpoint, "YYYY-MM-DD").format("MMM-DD"):
			moment(defaultCheckpoint, "YYYY-MM-DD").format("MMM-DD"),
		(b.file.frontmatter.deadline) ?
			moment(b.file.frontmatter.deadline, "YYYY-MM-DD").format("MMM-DD"):
			moment(defaultDeadline, "YYYY-MM-DD").format("MMM-DD"),
		]
	)
)
```


## Actions
```dataviewjs
const currentPerson = dv.current().file.name;
const defaultAccountablePerson = 'Vladimir Nosenko';
const defaultResponsiblePerson = 'Vladimir Nosenko';

const today = moment();
const tag_id = "#" + "action";
const pages =
	dv.pages(tag_id)
		.map(p => p.file.path);

async function pagesWithActionsRead(filePath) {
	var output = [];
	const content = await dv.io.load(filePath);
	const lines = content.split('\n');

	for (const line of lines) {
		if (line.trim() === '') continue;
		if (!line.includes(tag_id)) continue;
		output.push(
		{
			page: filePath,
			data: line.replace(tag_id,'')
		});
	}
		    
	return output;
}

var existingActions = [];
const actionsPromises = pages.map(page => pagesWithActionsRead(page));
const actionsArray = await Promise.all(actionsPromises);

function extractAndValidateDate(str) {
    const dateRegex = /\b\d{4}-\d{2}-\d{2}\b/;
    const match = str.match(dateRegex);
    if (match) {
        const date = match[0];
        if (moment(date, "YYYY-MM-DD", true).isValid()) {
            return date;
        }
    }
    return null;
}

actionsArray.forEach(actions => {
    actions.forEach(action => {
        var lines = action.data.toString().split(';');
        const filePath = action.page;
        let date = extractAndValidateDate(filePath);

        var comment = '';
        var status = '';
        var responsible = '';
        var accountable = '';

        lines.forEach(line => {
            line = line.trim();
            if (line.includes('- [ ]')) {
            	status = "in progress";
            	line = line.replace('- [ ]','').trim();
            	date = extractAndValidateDate(line);
            	if (date == null)
	            	comment = comment.trim() + ' ' + line.trim();
	        } else if (line.includes('- [x]')) {
		        status = "done";
		        line = line.replace('- [x]','').trim();
            	date = extractAndValidateDate(line);
            	if (date == null)
	            	comment = comment.trim() + ' ' + line.trim();
		    } else if (line.includes('acc:')) {
				accountable = line.replace('acc:','').trim();
			} else if (line.includes('resp:')) {
				responsible = line.replace('resp:','').trim();
            } else {
                comment = comment.trim() + ' ' + line.trim();
                comment = comment.trim();
            }
        });

		if (accountable == '') accountable = defaultAccountablePerson;
		if (responsible == '') responsible = defaultResponsiblePerson;
		if (responsible == currentPerson || accountable == currentPerson)
			existingActions.push({
				done: status,
				date: date,
				comment: comment,
				accountable: accountable,
				responsible: responsible,
				page: filePath
			});
    });
});

var responsibleExistingActions = existingActions;
//.filter(
//	  action => action.accountable != currentPerson
//);

if (responsibleExistingActions.length > 0) {
	dv.header(3, "Resposibilities");
	dv.table(["status", "date", "comment", "accountable", "page"],
		responsibleExistingActions.filter(action =>
			action.accountable != currentPerson)
	    .map((row) => [
		    row.done,
		    row.date,
		    row.comment,
		    "[["+row.accountable+"]]",
	        "[["+row.page+"]]"
	    ])
	);
} else {
	dv.header(3, "Here is no my responsibilities");
}

const accountableExistingActions = existingActions
.filter(
	action => action.accountable === currentPerson
);

if (accountableExistingActions.length > 0) {
	dv.header(3, "Accountables");
	dv.table(["status", "date", "comment", "responsible", "page"],
		responsibleExistingActions
	    .map((row) => [
		    row.done,
		    row.date,
		    row.comment,
		    "[["+row.responsible+"]]",
	        "[["+row.page+"]]"
	    ])
	);
} else {
	dv.header(3, "Here is no my accountables");
}

```

### Notes
```dataviewjs
// Prompt for filename
const full_tag_id = '[[' + dv.current().file.folder + '/' + dv.current().file.name + ']]'; // Tag is the current file title
const short_tag_id = '[[' + dv.current().file.name + ']]'; // Tag is the current file title

// Get all pages from /Journal folders
const pages = dv.pages('"Journal"')
    .map(p => p.file);

async function pagesWithActionsRead(pageFile) {
    var output = [];
    const content = await dv.io.load(pageFile.path);
    const lines = content.split('\n');
    let inCodeBlock = false;
    let codeBlock = [];
    let inHeaderSection = false;
    let headerSection = [];
    let startHeaderLevel = 0;
    let consecutiveNewLines = 0;

    for (const line of lines) {
        if (inCodeBlock) {
            codeBlock.push(line);
            if (line.trim() === '```') {
                inCodeBlock = false;
                output.push({
                    page: pageFile.link,
                    data: codeBlock.join('\n'),
                    mtime: pageFile.mtime
                });
                codeBlock = [];
            }
            continue;
        }

        if (inHeaderSection) {
            if (line.startsWith('#')) {
                const currentHeaderLevel = line.split(' ')[0].length;
                if (currentHeaderLevel <= startHeaderLevel) {
                    inHeaderSection = false;
                    output.push({
                        page: pageFile.link,
                        data: headerSection.join('\n'),
                        mtime: pageFile.mtime
                    });
                    headerSection = [];
                }
            }
            if (line.trim() === '') {
                consecutiveNewLines++;
                if (consecutiveNewLines >= 2) {
                    inHeaderSection = false;
                    output.push({
                        page: pageFile.link,
                        data: headerSection.join('\n'),
                        mtime: pageFile.mtime
                    });
                    headerSection = [];
                }
            } else {
                consecutiveNewLines = 0;
            }
            if (inHeaderSection) {
                headerSection.push(line);
            }
            continue;
        }

        if (line.includes(full_tag_id) || line.includes(short_tag_id)) {
            if (line.startsWith('#')) {
                startHeaderLevel = line.split(' ')[0].length;
                inHeaderSection = true;
                headerSection.push(line);
            } else if (line.trim().startsWith('```')) {
                inCodeBlock = true;
                codeBlock.push(line);
            } else {
                output.push({
                    page: pageFile.link,
                    data: line,
                    mtime: pageFile.mtime
                });
            }
        }
    }

    // Push any remaining header section if the file ends without encountering a lower or same level header or two new lines
    if (headerSection.length > 0) {
        output.push({
            page: pageFile.link,
            data: headerSection.join('\n'),
            mtime: pageFile.mtime
        });
    }

    return output;
}

const actionsPromises = pages.map(page => pagesWithActionsRead(page));
const actionsArray = await Promise.all(actionsPromises);

// Flatten the array of arrays and sort by modification time
const flattenedActions = actionsArray.flat(); //.sort((a, b) => b.mtime - a.mtime);

// Read the current file content
const currentFilePath = dv.current().file.path;
let currentFileContent = await dv.io.load(currentFilePath);
let currentFileLines = currentFileContent.split('\n');

flattenedActions.forEach(action => {
    let actionData = action.data.replace(full_tag_id,'').replace(short_tag_id,'');
    let actionPageLink = action.page; 
    const actionText = `##### ${actionPageLink}:\n${actionData}\n\n---`;
    const compText =
        actionData
            .replace('- [ ] ', '')
            .replace('- [x] ', '');
    console.log("compText: " + compText);
    if (!currentFileContent.includes(compText)) {
        currentFileLines.push(actionText);
    }
});


// Write the updated content back to the current file using Obsidian API
await app.vault.modify(app.vault.getAbstractFileByPath(currentFilePath), currentFileLines.join('\n'));
```
---
