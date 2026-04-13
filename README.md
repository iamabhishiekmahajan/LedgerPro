// --- MAIN FUNCTIONS ---

function doGet(e) {
  const action = e.parameter.action;
  const callback = e.parameter.callback;
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  
  if (action === 'getData') {
    const txns = getRowsData(getOrCreateSheet(ss, "Transactions"));
    const customers = getCustomerMap(getOrCreateSheet(ss, "Customers"));
    
    const result = JSON.stringify({ customers, txns });
    return ContentService.createTextOutput(callback + "(" + result + ")")
      .setMimeType(ContentService.MimeType.JAVASCRIPT);
  }
}

function doPost(e) {
  const ss = SpreadsheetApp.getActiveSpreadsheet();
  const sheet = getOrCreateSheet(ss, "Transactions");
  const custSheet = getOrCreateSheet(ss, "Customers");
  const data = JSON.parse(e.postData.contents);

  if (data.action === 'addTxn') {
    sheet.appendRow([
      data.txn.id, 
      data.txn.name, 
      data.txn.amount, 
      "'" + data.txn.date, // Force String with single quote
      data.txn.type, 
      data.txn.remarks || ""
    ]);
    updateCustomer(custSheet, data.customer);
    return ContentService.createTextOutput("Success").setMimeType(ContentService.MimeType.TEXT);
  }
}

// --- HELPER FUNCTIONS ---

function getOrCreateSheet(ss, name) {
  let sheet = ss.getSheetByName(name);
  if (!sheet) { 
    sheet = ss.insertSheet(name);
    if(name === "Transactions") sheet.appendRow(["ID", "Name", "Amount", "Date", "Type", "Remarks"]);
    if(name === "Customers") sheet.appendRow(["Name", "Phone", "JoinedDate", "City"]);
  }
  return sheet;
}

function getRowsData(sheet) {
  const data = sheet.getDataRange().getValues();
  if (data.length <= 1) return []; 
  
  const rows = data.slice(1);
  return rows.map(r => {
    let dateStr = "";
    if (r[3] instanceof Date) {
      // Direct IST formatting in Backend
      dateStr = Utilities.formatDate(r[3], "GMT+5:30", "yyyy-MM-dd");
    } else {
      dateStr = String(r[3]).replace(/'/g, "").trim();
    }
    return {
      id: r[0],
      name: r[1],
      amount: Number(r[2]),
      date: dateStr, 
      type: r[4],
      remarks: r[5] || ""
    };
  });
}

function getCustomerMap(sheet) {
  const data = sheet.getDataRange().getValues();
  const map = {};
  if (data.length <= 1) return map;

  const rows = data.slice(1);
