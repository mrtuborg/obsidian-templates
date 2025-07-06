[[ Invalid date ]] | [[ Invalid date ]]

## Resource: :luc_banknote: MONEY
### Recurrent payments for this month
```dataviewjs
const tag_id = "#recurrence-generator";
const pages =
	dv.pages(tag_id)
		.map(p => p.file.path);

async function pagesGeneratorsRead(filePath) {
	var output = [];
	const content = await dv.io.load(filePath);
	const lines = content.split('\n');

	for (const line of lines) {
		if (line.trim() === '') continue;
		if (!line.includes(tag_id)) continue;
		output.push({
			page: filePath,
			data: line.replace(tag_id,'')
			});
	}
		    
	return output;
}

const recurrence_regex = /\((\d+)\)/;
function isMoneyAmount(str) {
	const regex = /^([A-Za-z]{3}\s?)?[-+]?\d+(\.\d{1,2})?(\s?[A-Za-z]{3})?$/;
	return regex.test(str);
}

const curMonth = moment(dv.current().file.name, "YYYY-MM").format("MM");
const curYear = moment(dv.current().file.name, "YYYY-MM").format("YYYY");

var recurrences = [];
const generatorPromises = pages.map(page => pagesGeneratorsRead(page));
const generatorsArray = await Promise.all(generatorPromises);

generatorsArray.forEach(generators => {
    generators.forEach(generator => {
        var lines = generator.data.toString().split(';');

        var deadline = 0;
        var moneyAmount = 0;
        var srcString = '';
        var dstString = '';
        var comment = '';

        lines.forEach(line => {
            line = line.trim();
            if (line.includes('monthly')) {
                const match = line.match(recurrence_regex);
                if (match) {
                    deadline = moment(curYear + "-" + curMonth + "-" + match[1], "YYYY-MM-DD").format("YYYY-MM-DD");
                }
            } else if (line.includes('src')) {
                srcString = line.replace('src:', '');
            } else if (line.includes('dst')) {
                dstString = line.replace('dst:', '');
            } else if (isMoneyAmount(line)) {
                moneyAmount = line;
            } else {
                comment = line;
            }
        });

        recurrences.push({
            date: deadline,
            amount: moneyAmount,
            src: srcString,
            dst: dstString,
            comment: comment,
            page: generator.page
        });
    });
});

dv.table(["deadline", "amount", "src", "dst", "comment", "page"],
    recurrences.map((row) => [
        row.date,
        row.amount,
        row.src,
        row.dst,
        row.comment,
        '[['+row.page+']]'
    ])
);

// Function to create or append to the daily note
async function createOrAppendDailyNote(formattedDate, contentToAppend) {
	// Define the path for the daily note
	const dailyNotePath = `Journal/${formattedDate}.md`;
	const fileExists = await app.vault.adapter.exists(dailyNotePath);
	
	if (!fileExists) { // Create a new daily note
		await app.vault.create(dailyNotePath, `# ${formattedDate}\n`);
	}

	// Append content to the daily note
	const file = await app.vault.getAbstractFileByPath(dailyNotePath);
	const data = await app.vault.read(file);
	
	// We need to add action only if it missing for that day.
	if (!data.includes(contentToAppend)) {
		console.log("note " + dailyNotePath + " is missing this action");
		await app.vault.modify(file,
			data +
			"\n- [ ] " + contentToAppend +
			" #expenses");
	}
}


for (var r = 0; r < recurrences.length; r++) {
	// Run the function
	const generatedAction =
		 recurrences[r].amount  + "; " +
		 recurrences[r].comment + "; from: " +
		 recurrences[r].src     + "; to: " +
		 recurrences[r].dst;

	createOrAppendDailyNote(recurrences[r].date, generatedAction).then(() => {
		console.log('Daily note created or updated successfully.');
	}).catch(err => {
		console.error('Error creating or updating daily note:', err);
	});
}


```

### Actual payments made:
```dataviewjs
const tag_id = "#expenses";
const pages =
	dv.pages(tag_id)
		.map(p => p.file.path);

async function pagesGeneratorsRead(filePath) {
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

function parseMoneyAmount(str) {
    const regex = /^([A-Za-z]{3}\s?)?([-+]?\d+(\.\d{1,2})?)(\s?[A-Za-z]{3})?$/;
    const match = str.match(regex);
    
    if (match) {
        const currencyPrefix = match[1] ? match[1].trim() : null;
        const amount = match[2];
        const currencySuffix = match[4] ? match[4].trim() : null;
        
        // Determine the currency
        const currency = currencyPrefix || currencySuffix || 'NOK';
        
        return {
            amount: parseFloat(amount),
            currency: currency
        };
    }
    
    return 0;
}

var existingPayments = [];
const expensesPromises = pages.map(page => pagesGeneratorsRead(page));
const expensesArray = await Promise.all(expensesPromises);

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

expensesArray.forEach(expenses => {
    expenses.forEach(payment => {
        var lines = payment.data.toString().split(';');
        const filePath = payment.page;
        const date = extractAndValidateDate(filePath); 
        var moneyAmount = 0;
        var expense='';
        var currency = 0;
        var srcString = '';
        var dstString = '';
        var comment = '';
        var status = '';

        lines.forEach(line => {
            line = line.trim();
            if (line.includes('- [ ]')) {
            	status = "unpaid";
            	line = line.replace('- [ ]','').trim();
				({ amount: moneyAmount, currency: currency } = parseMoneyAmount(line));
	        } else if (line.includes('- [x]')) {
		        status = "paid";
		        line = line.replace('- [x]','').trim();
				({ amount: moneyAmount, currency: currency } = parseMoneyAmount(line));
            } else if (line.includes('from:')) {
				expense = 'Expenses:' + line.replace('from:','').trim();
            } else {
                comment = comment.trim() + ' ' + line.trim();
                comment = comment.trim();
            }
        });

        existingPayments.push({
	        done: status,
			date: date,
			comment: comment,
			expense: expense,
            amount: moneyAmount,
			currency: currency,
            page: filePath
        });
        
    });
});

const curMonth = moment(dv.current().file.name, "YYYY-MM").format("MM");
const curYear = moment(dv.current().file.name, "YYYY-MM").format("YYYY");

console.log(curYear + '-' + curMonth);

dv.table(["status", "date", "comment", "expense", "amount", "currency", "page"],
	existingPayments
	.filter((row) =>
		moment(row.date,"YYYY-MM-DD").format("YYYY-MM") ==
		moment(curYear + '-' + curMonth, "YYYY-MM").format("YYYY-MM")
	)
    .map((row) => [
	    row.done,
	    row.date,
	    row.comment,
	    row.expense,
        row.amount,
        row.currency,
        "[["+row.page+"]]"
    ])
);

// Обновление журнала транзакций
const journalFilePath =
	'/Resources/Money/' + curYear +
	'/journal-test-' + curYear + '-' + curMonth + '.md';

const fs = require('fs');
function app_add(filePath, contentArray) {
	let content = contentArray.toString();
	fs.appendFileSync(app.vault.adapter.basePath + filePath, content, function(err) {
		if (err) {
			console.log(err);
		} else {
			console.log("All good...");
	    }
    });
}

async function fileDataRead() {	
    let data = [];
    try {
        const content = await dv.io.load(journalFilePath);
        if (!content) {
            console.warn("File is empty or could not be loaded.");
            return data; // Return empty array if file is empty
        }

        const lines = content.trim().split(/\n{2,}/);
        if (lines.length === 0) {
            console.warn("File format is incorrect or no valid data found.");
            return data; // Return empty array if no valid data
        }

        data = lines.map(line => {
            const [dateLocation, transaction] = line.split('\n').map(part => part.trim());
            if (!dateLocation || !transaction) {
                console.warn("Invalid line format:", line);
                return { date: '', place: '', expense: '', amount: '', currency: '' };
            }

            const [date, comment] = dateLocation.split(' * ').map(part => part.trim());
            const [expense, amount, currency] = transaction.split(' ').map(part => part.trim());
            return { 
                date: date || '', 
                comment: comment || '', 
                expense: expense || '', 
                amount: parseInt(amount) || '', 
                currency: currency || '' 
            };
        });
    } catch (error) {
        console.error("Error reading file:", error);
    }
    return data;
}

const categories = ["done", "date", "comment", "expense", "amount", "currency"]
const fileData = await fileDataRead();

//console.log(fileData);
//console.log(existingPayments);

var newDataToSave = [];

newDataToSave =
	existingPayments.filter(payment => 
	  payment.done === 'paid' &&
	  !fileData.some(transaction => 
		transaction.date === payment.date &&
		transaction.comment === payment.comment &&
		transaction.expense === payment.expense &&
		transaction.amount === payment.amount &&
		transaction.currency === payment.currency
	  )
	);

//console.log(newDataToSave);

if (newDataToSave && newDataToSave.length) {
	dv.header(2, "Expenses not yet added to transasctions journal:");
	dv.table(categories, newDataToSave.map(d => [d.done, d.date, d.comment, d.expense, d.amount, d.currency]));
	
	newDataToSave.map(d => {
			const transaction1stRow = '\n' + d.date + ' * ' + d.comment;
			const transaction2ndRow = '\n' + d.expense + ' ' + d.amount + ' ' + d.currency + '\n';
			console.log("new transaction for journal: " + transaction1stRow);
			app_add(journalFilePath, transaction1stRow);
			app_add(journalFilePath, transaction2ndRow);
		});
	}
	
```

## Resources: :obs_calendar_glyph: TIME
### Month goals
```dataviewjs
const curMonth = moment(dv.current().file.name, "YYYY-MM").format("YYYY-MM");
const curYear = moment(dv.current().file.name, "YYYY-MM").format("YYYY");

const today = moment(new Date());

let defaultCheckpoint = today;
let defaultDeadline = today;

function monthNumber(page, type) {
    // Validate input parameters
    if (!page || (!page.name && !page.frontmatter)) {
        console.error("Invalid page data provided to weekNumber function.");
        return curMonth;
    }

    let dateStr;
    if (moment(page.name, "YYYY-MM-DD", true).isValid()) {
        dateStr = page.name;
    } else if (type === "deadline" && page.frontmatter.deadline) {
        dateStr = page.frontmatter.deadline;
    } else if (page.frontmatter.checkpoint) {
        dateStr = page.frontmatter.checkpoint;
    } else {
        console.error("No valid date found for weekNumber calculation for " + page.name + " and type " + type);
        dateStr = today;
    }

    return moment(dateStr, "YYYY-MM-DD").format("YYYY-MM");
}

dv.table(["stage", "title", "checkpoint", "deadline"],
	dv.pages('"Activities"')
	.where(b => 
		(b.file.frontmatter) &&
		(b.file.frontmatter.stage != "done") &&
		((curMonth == monthNumber(b.file, "deadline")) ||
		 (curMonth == monthNumber(b.file, "checkpoint")))
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


## Resources: :luc_phone_call: PEOPLE
```dataviewjs
const currentPerson = "Vladimir Nosenko";
const tag_id = "#action";
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
        const date = extractAndValidateDate(filePath);

        var comment = '';
        var status = '';
        var responsible = '';
        var accountable = '';

        lines.forEach(line => {
            line = line.trim();
            if (line.includes('- [ ]')) {
            	status = "in progress";
            	line = line.replace('- [ ]','').trim();
	        } else if (line.includes('- [x]')) {
		        status = "done";
		        line = line.replace('- [x]','').trim();
		    } else if (line.includes('acc:')) {
				accountable = line.replace('acc:','').trim();
			} else if (line.includes('resp:')) {
				responsible = line.replace('resp:','').trim();
            } else {
                comment = comment.trim() + ' ' + line.trim();
                comment = comment.trim();
            }
        });

		if (accountable == '') accountable = currentPerson;
		if (responsible == '') responsible = currentPerson;
	
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

dv.header(3, "My resposibilities");
dv.table(["status", "date", "comment", "accountable", "page"],
    existingActions
    .filter(action => 
	  action.responsible === currentPerson)
    .map((row) => [
	    row.done,
	    row.date,
	    row.comment,
	    "[["+row.accountable+"]]",
        "[["+row.page+"]]"
    ])
);

dv.header(3, "My accountables");
dv.table(["status", "date", "comment", "responsible", "page"],
    existingActions
    .filter(action => 
	  action.accountable === currentPerson)
    .map((row) => [
	    row.done,
	    row.date,
	    row.comment,
	    "[["+row.responsible+"]]",
        "[["+row.page+"]]"
    ])
);


```
