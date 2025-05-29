## Copy to clipboard

```js
const copyToClipboard = (txt) => {
  try {
    navigator.clipboard.writeText(url);
  } catch (e) {
    unsecuredCopyToClipboard(txt);
  }
  showToastNotification("Copied to clipboard", "success");
};

const unsecuredCopyToClipboard = (text) => {
  const ta = document.createElement("textarea");
  ta.value = text;
  document.body.appendChild(ta);
  ta.focus();
  ta.select();
  try {
    document.execCommand("copy");
  } catch (err) {
    console.error("Unable to copy to clipboard", err);
  }
  document.body.removeChild(ta);
};
```

## fetch

```js
const fetchExample = (url) => {
  fetch(url, {
    method: "POST",
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
    },
    body: JSON.stringify({ ... }),
  })
    .then((res) => {
      if (res.ok) {
        return res.json();
      }
      return Promise.reject(res);
    })
    .then((body) => {
      ...
    })
    .catch((error) => {
      ...
    })
    .finally(() => {
     ...
    });
};
```

## Download blob/file with fetch

```js
const downloadFile = (fileUrl) => {
  fetch(fileUrl, {
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
    },
  })
    .then((response) => {
      if (!response.ok) {
        return response.json().then((body) => {
          // error handling
          if (body.status == "error") {
            return downloadErrorHandler();
          }
          throw new Error("HTTP status " + response.status);
        });
      }
      return response.blob().then((blob) => {
        filename = extractFilenameFromHeaders(response.headers);
        return downloadBlob(blob, filename);
      });
    })
    .catch((error) => {
      console.log(error);
    });
};

const downloadBlob = (blob, filename) => {
  const link = document.createElement("a");
  link.href = window.URL.createObjectURL(blob);
  link.download = filename;
  link.click();
  link.remove();
};

// Extract filename from headers
const extractFilenameFromHeaders = (headers) => {
  const header = headers.get("Content-Disposition");
  const parts = header.split(";");
  return parts[1].split("=")[1];
};
```

## Open url on same tab

```js
window.location.href = url;
```

## Open url on new tab

```js
window.open(url, "_blank");
```

## Axios [Link](https://axios-http.com/)

```js
axios
  .get("/url", {
    headers: { "Content-Type": "application/json" },
  })
  .then((response) => {
    loadData(response.data);
    ...
  })
  .catch(function (error) {
    console.error(error);
  });
```

## Abort Controller

```js
let abortController = new AbortController();


const abortRequestAndResetController = (controller) => {
  if (controller) {
    controller.abort();
  }

  return new AbortController();
};

// For Bootstrap 5 modal on open/close
const modal = document.getElementById("modal_id");
modal.addEventListener("hidden.bs.modal", () => {
  abortController.abort();
});

modal.addEventListener("show.bs.modal", () => {
  abortController = new AbortController();
});


// Fetch
fetch("/url", {
    method: "GET",
    headers: {
      Accept: "application/json",
      "Content-Type": "application/json",
    },
    body: JSON.stringify(params),
    signal: abortController.signal,
  })
    .then((res) => res.json())
    .then((data) => {
      ...
    })
    .catch((error) => {
      if (error.name === "AbortError" || error.name === "CanceledError") {
        console.log("Fetch aborted");
      } else {
        console.log(error);
      }
    });


// Axios
axios
  .get("/url", {
    headers: { "Content-Type": "application/json" },
  })
  .then((response) => {
    loadData(response.data);
    ...
  })
  .catch(function (error) {
    console.error(error);
  });

```

## Simple table class and manager

```js
class SimpleTable {
  constructor(containerId, tableId, columnsConfig, rows, cssClasses) {
    this.containerId = containerId;
    this.tableId = tableId;
    this.columnsConfig = columnsConfig;
    this.rows = rows;
    this.cssClasses = cssClasses;
  }

  getData = () => {
    return this.rows;
  };

  getTextAlign = (hozAlign) => {
    if (!hozAlign) {
      return "";
    }

    switch (hozAlign) {
      case "right":
        return "text-end";

      case "left":
        return "text-start";

      default:
        `text-${hozAlign}`;
    }
  };

  generateTHead = () => {
    const thRows = this.generateThRows(this.columnsConfig);

    return `<thead>
      ${thRows.join("")}
    </thead>`;
  };

  generateThRows = (columnsConfig, currentLevel = 0) => {
    const rows = [];
    const currentRow = [];

    columnsConfig.forEach((columnConfig) => {
      const textAlign = this.getTextAlign(columnConfig.hozAlign);
      const fieldName = columnConfig.field ? columnConfig.field : "";
      const rowSpan = columnConfig.headerRowSpan
        ? `rowspan="${columnConfig.headerRowSpan}"`
        : "";

      if (columnConfig.columns) {
        currentRow.push(`<th class="${fieldName} ${textAlign}" ${rowSpan} colspan="${columnConfig.columns.length}">
                          ${columnConfig.title}</th>`);

        const nestedThRows = this.generateThRows(
          columnConfig.columns,
          currentLevel + 1
        );
        rows.push(...nestedThRows);
      } else {
        currentRow.push(
          `<th class="${fieldName} ${textAlign}" ${rowSpan}>${columnConfig.title}</th>`
        );
      }
    });

    rows.unshift(`<tr>${currentRow.join("")}</tr>`);
    return rows;
  };

  generateTr = (rowData) => {
    const tds = this.columnsConfig
      .map((columnConfig) => {
        if (columnConfig.columns) {
          return columnConfig.columns
            .map((nestedColumnConfig) => {
              return this.generateTd(
                rowData[nestedColumnConfig.field],
                nestedColumnConfig
              );
            })
            .join("");
        } else {
          return this.generateTd(rowData[columnConfig.field], columnConfig);
        }
      })
      .join("");

    return `<tr>${tds}</tr>`;
  };

  generateTBody = () => {
    if (!this.rows?.length) {
      const colspan = this.columnsConfig.length;
      return `<tbody>
        <tr><td colspan="${colspan}">No data available</td></tr>
      </tbody>`;
    }

    const trs = this.rows
      .map((rowData) => {
        return this.generateTr(rowData);
      })
      .join("");

    return `<tbody>${trs}</tbody>`;
  };

  generateTd = (cellData, columnConfig) => {
    const textAlign = this.getTextAlign(columnConfig.hozAlign);
    const styles = columnConfig.styles ? columnConfig.styles.join(" ") : "";
    const value = valFormatter(
      cellData,
      columnConfig.formatter,
      columnConfig.formatterParams
    );
    return `<td class="${columnConfig.field} ${textAlign} ${styles}">${value}</td>`;
  };

  render = () => {
    const container = document.getElementById(this.containerId);
    if (container) {
      const tableClasses = this.cssClasses.tableStyle
        ? this.cssClasses.tableStyle
        : "";
      container.innerHTML = `
      <table id="${this.tableId}" class="table ${tableClasses}">
        ${this.generateTHead()}
        ${this.generateTBody()}
      </table>
      `;
    }
  };

  refreshAndRender = (data) => {
    this.rows = data;
    this.render();
  };
}

class SimpleTableManager {
  constructor() {
    this.tables = {};
  }

  create = (containerId, tableId, columns, rows, cssClasses) => {
    if (this.tables[tableId]) {
      console.error(`Table ${tableId} already exists`);
      return;
    }

    const table = new SimpleTable(
      containerId,
      tableId,
      columns,
      rows,
      cssClasses
    );
    this.tables[tableId] = table;
    return table;
  };

  find = (tableId) => {
    const table = this.tables[tableId];
    if (!table) {
      return null;
    }
    return table;
  };

  find_or_create = (containerId, tableId, columns, rows, cssClasses) => {
    const table = this.find(tableId);
    if (table) {
      return table;
    }
    return this.create(containerId, tableId, columns, rows, cssClasses);
  };

  renderTable = (tableId) => {
    const table = this.find(tableId);
    if (table) {
      table.render();
    }
  };

  refreshTableData = (tableId, data) => {
    const table = this.find(tableId);
    if (table) {
      table.refreshAndRender(data);
    }
  };
}

export { SimpleTableManager, SimpleTable };
```
