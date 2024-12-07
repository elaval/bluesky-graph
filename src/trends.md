---
toc: false
sql:
  master: https://raw.githubusercontent.com/elaval/BskyFollowersTrend/refs/heads/main/master.parquet
  followers_log: https://raw.githubusercontent.com/elaval/BskyFollowersTrend/refs/heads/main/followers_log.parquet
---

<style type="text/css">

p,
table,
figure,
figcaption,
h1,
h2,
h3,
h4,
h5,
h6,
.katex-display {
  max-width: none;
}

</style>

```sql id=master 
-- Fetch the master data: Basic user information, including handle, followers_count, etc.
SELECT * FROM master
```

```sql id=followers_log 
-- Fetch the followers log data: Timestamps and follower counts over time.
SELECT * FROM followers_log
````

```js
const defaultUser = `elaval.bsky.social`
```

<div class="container">
  <div class="row justify-content-center">
    <div class="col-12">
    
# Número de seguidores en Bluesky
## Evolución en el tiempo

**Última actualización:** ${moment(_.chain([...master]).map(d => d.timestamp).max().value()).format("D MMM YYYY, HH:mm")}

¿Te interesa conocer cómo ha evolucionado el número de tus seguidores a lo largo del tiempo?  

He construido un proceso automático que, desde el 4 de diciembre de 2024, monitorea cuatro veces al día la cantidad de seguidores de un conjunto de cuentas en Bluesky y almacena estos datos. Esta página muestra cómo varía el número de seguidores de las cuentas monitoreadas a lo largo del tiempo.  

Por el momento, se incluyen las cuentas que me siguen ([@elaval.bsky.social](https://bsky.app/profile/elaval.bsky.social)). El monitoreo comenzó el 4 de diciembre de 2024 o el día en que la cuenta empezó a seguirme, si esto ocurrió después de esa fecha.  

Este proyecto es una exploración destinada a analizar el comportamiento de los seguidores en Bluesky y a probar distintos métodos para presentar y compartir la información. En el futuro, podrían realizarse modificaciones en la frecuencia de monitoreo o en los criterios utilizados para seleccionar las cuentas a seguir (por ahora, la intención es partir de una base sencilla).


```js
// Create an autocomplete widget for selecting a user by their handle.
const selectedUser = view(createAutocomplete(_.sortBy([...master], d => d.handle), {
  searchFields: ["handle", "displayName"],
  formatFn: d => `${d.handle} - ${d.displayName} (${d.followers_count} seguidores)`,
  defaultValue: defaultUser, // Change this to an actual handle from your data
  label: "Seleccione una cuenta en Bluesky",
  description: null//`Lista compuesta por quienes siguen a ${defaultUser}`
}));
```


<div class="container md-4">
  <div class="row row-cols row-cols-sm-2 row-cols-md-3 row-cols-lg-4 g-4">
    ${[userProfiles[selectedUser || defaultUser]]
      .map(d => html`
        <div class="col">
          <div class="card h-100 shadow-sm custom-card">
            <div class="d-flex justify-content-center mt-3">
              <img src="${d.avatar_url}" class="rounded-circle" alt="${d.handle}" style="width: 100px; height: 100px; object-fit: cover;">
            </div>
            <div class="card-body text-center">
              <h5 class="card-title">${d.displayName}</h5>
              <p class="card-text"><a href="https://bsky.app/profile/${d.handle}" target="_blank">@${d.handle}</a></p>
              <p class="card-text">Cuenta creada el ${moment(d.created_at).format("D MMM YYYY")}</p>
              <p class="card-text">
                <span class="badge bg-primary text-white">
                  Seguidores: ${d.followers_count}
                </span>
                <p class="small">
                  ${d.description}
                </p>
              </p>
            </div>
          </div>
        </div>
      `)
    }

  </div> 
</div>

<style>
  .custom-card {
    max-width: 300px;
    margin: 0 auto; /* Center align the card in its column */
  }
  .custom-card .badge {
    background-color: #007bff; /* Primary background */
    color: white; /* White text color */
  }
</style>


<div class="card">
${chart}
</div>

  </div>
</div>

```js
const toggleInicio = view(Inputs.toggle({
  label: "Escala de tiempo desde que la cuenta fue creada",
  value: false
}))
```

```js
// Group the followers_log data by handle for quicker lookups.
const followersDataRaw = _.chain([...followers_log])
  .groupBy(d => d.handle)
  .value();
```

```js
// Initialize a profiles object to store user profiles keyed by their handle.
const userProfiles = {};
```

```js
// Populate userProfiles for quick access by handle
[...master].forEach((d) => {
  userProfiles[d.handle] = d;
});

```

```js

// Build the chart for the selected user
const chart = (() => {
  // Get and sort the user's followers data over time
  const dataPlot = selectedUser && _.chain(followersDataRaw[selectedUser])
    .sortBy(d => d.timestamp)
    .value() || [];

  // Initial data includes the user's creation date and their first recorded follower count
  const initialData = !toggleInicio ? [] : selectedUser && _.concat(
    {
      timestamp: new Date(userProfiles[selectedUser].created_at),
      followers_count: 0
    },
    _.chain(followersDataRaw[selectedUser])
      .sortBy(d => d.timestamp)
      .first()
      .value()
  ) || [];

  return Plot.plot({
    title: `Evolución de número de seguidores de ${selectedUser}`,
    subtitle: toggleInicio && `La línea roja punteada es una proyección desde el momento de creación de la cuenta hasta el inicio del monitoreo.` || `La escala de tiempo se inicia cuando comenzó el monitoreo para esta cuenta`,
    caption: `Fuente de datos: Datos recolectados por @elava.bsky.social utilizano API de Bluesahy (repositorio https://github.com/elaval/BskyFollowersTrend)`,
    height: 500,
    x: { type: "time", label:"Fecha" },
    y: { tickFormat: "d", grid: true, type: "linear" , zero:true, label:"# de seguidores"},
    marks: [
      Plot.lineY(initialData, {
        x: "timestamp",
        y: (d) => +d["followers_count"],
        stroke: (d) => "red",
        strokeDasharray: "10"
      }),
      Plot.lineY(dataPlot, {
        x: d => moment(d.timestamp).toDate(),
        y: d => +d.followers_count,
        stroke: "handle"
      }),
      Plot.dot(dataPlot, {
        x: d => moment(d.timestamp).toDate(),
        r: 1.5,
        y: d => +d.followers_count,
        fill: "handle",
        tip: true,
        title: d => `${moment.utc(d.timestamp).format("DD MMM YYY HH:mm")}\n${d3.format("d")(d.followers_count)} seguidores`
      })
    ]
  });
})();
```

```js

function createAutocomplete_old(items = [], {
  searchFields = ["handle", "displayName", "description"],
  formatFn = d => d.handle,
  defaultValue = null,
  label = null,
  description = null
} = {}) {
  const container = document.createElement("div");
  container.style.position = "relative";
  container.style.width = "500px";
  container.style.fontFamily = "sans-serif";
  container.value = ""; 

  // If a label is provided, create a label element
  if (label) {
    const labelEl = document.createElement("label");
    labelEl.textContent = label;
    labelEl.style.display = "block";
    labelEl.style.marginBottom = "4px";
    labelEl.style.fontWeight = "bold";
    container.appendChild(labelEl);
  }

  // If a description is provided, create a description element
  if (description) {
    const descEl = document.createElement("div");
    descEl.textContent = description;
    descEl.style.fontSize = "0.9em";
    descEl.style.color = "#555";
    descEl.style.marginBottom = "8px";
    container.appendChild(descEl);
  }

  const input = document.createElement("input");
  input.type = "text";
  input.style.width = "100%";
  input.style.boxSizing = "border-box";
  input.placeholder = "Type to search...";
  input.style.padding = "8px";
  input.style.border = "1px solid #ccc";
  input.style.borderRadius = "4px";
  container.appendChild(input);

  const dropdown = document.createElement("div");
  dropdown.style.position = "absolute";
  dropdown.style.width = "100%";
  dropdown.style.boxSizing = "border-box";
  dropdown.style.border = "1px solid #ccc";
  dropdown.style.borderTop = "none";
  dropdown.style.background = "#fff";
  dropdown.style.zIndex = "999";
  dropdown.style.display = "none";
  dropdown.style.maxHeight = "300px"; 
  dropdown.style.overflowY = "auto";
  dropdown.style.borderRadius = "0 0 4px 4px";
  dropdown.style.whiteSpace = "nowrap";
  container.appendChild(dropdown);

  // Set default value if provided
  if (defaultValue) {
    input.value = defaultValue;
    container.value = defaultValue;
    container.dispatchEvent(new Event("input"));
  }

  function clearDropdown() {
    dropdown.innerHTML = "";
    dropdown.style.display = "none";
  }

  function updateDropdown(value) {
    clearDropdown();
    if (!value) return;

    const lowerVal = value.toLowerCase();
    const filtered = items
      .filter(d => searchFields.some(field => {
        const fieldValue = d[field];
        return fieldValue && fieldValue.toString().toLowerCase().includes(lowerVal);
      }))
      .slice(0, 10);

    if (filtered.length > 0) {
      filtered.forEach(d => {
        const option = document.createElement("div");
        option.style.padding = "8px";
        option.style.cursor = "pointer";
        option.style.borderBottom = "1px solid #eee";
        option.style.whiteSpace = "nowrap";
        option.textContent = formatFn(d);

        option.addEventListener("mousedown", () => {
          // After selection, show only the handle in the input
          input.value = d.handle;
          container.value = d.handle;
          container.dispatchEvent(new Event("input"));
          clearDropdown();
        });

        option.addEventListener("mouseover", () => {
          option.style.background = "#f0f0f0";
        });

        option.addEventListener("mouseout", () => {
          option.style.background = "#fff";
        });

        dropdown.appendChild(option);
      });

      dropdown.style.display = "block";
    }
  }

  // On focus, just clear the input text to start a new search, but do not clear container.value.
  // This ensures the previously selected user remains in container.value if no new selection is made.
  input.addEventListener("focus", () => {
    // Clear the visible text, but keep the currently selected value in container.value stable
    input.value = "";
  });

  input.addEventListener("input", () => {
    const val = input.value.trim();
    updateDropdown(val);
  });

  input.addEventListener("blur", () => {
    setTimeout(clearDropdown, 150);
  });

  return container;
}

function createAutocomplete(items = [], {
  searchFields = ["handle", "displayName", "description"],
  formatFn = d => `${d.handle} - ${d.displayName} (${d.description})`,
  defaultValue = null,
  label = null,
  description = null,
  placeholderText = "Escribe para buscar...",
  fetchAutoSuggest = async (value) => {
    // Simulate fetching filtered results
    return items.filter(d => searchFields.some(field => {
      const fieldValue = d[field];
      return fieldValue && fieldValue.toString().toLowerCase().includes(value.toLowerCase());
    })).slice(0, 10); // Limit results
  }
} = {}) {
  // Create the form container
  const form = document.createElement("form");
  form.style.position = "relative";
  form.style.width = "100%";
  form.style.fontFamily = "sans-serif";
  form.onsubmit = (event) => event.preventDefault();

  // Add label if provided
  if (label) {
    const labelEl = document.createElement("label");
    labelEl.textContent = label;
    labelEl.style.display = "block";
    labelEl.style.marginBottom = "4px";
    labelEl.style.fontWeight = "bold";
    form.appendChild(labelEl);
  }

  // Add description if provided
  if (description) {
    const descEl = document.createElement("div");
    descEl.textContent = description;
    descEl.style.fontSize = "0.9em";
    descEl.style.color = "#555";
    descEl.style.marginBottom = "8px";
    form.appendChild(descEl);
  }

  // Create the input field
  const input = document.createElement("input");
  input.type = "text";
  input.placeholder = placeholderText;
  input.style.width = "100%";
  input.style.boxSizing = "border-box";
  input.style.padding = "8px";
  input.style.border = "1px solid #ccc";
  input.style.borderRadius = "4px";
  input.value = defaultValue || "";
  form.appendChild(input);

  // Create the datalist for suggestions
  const datalist = document.createElement("datalist");
  datalist.id = `autosuggest-results-${Math.random().toString(36).substr(2, 9)}`;
  input.setAttribute("list", datalist.id);
  form.appendChild(datalist);

  // Store the current value
  form.value = defaultValue || "";

  // Handle input change (selection made from the dropdown)
  input.onchange = (event) => {
    const value = event.target.value;

    // Find the selected item and set the handle only
    const selectedItem = items.find(d => formatFn(d) === value);
    if (selectedItem) {
      form.value = selectedItem.handle; // Set the handle as the value
      input.value = selectedItem.handle; // Update input to show only the handle
    } else {
      form.value = value; // Fallback to raw input value
    }

    input.blur(); // Dismiss keyboard (mobile)
    form.dispatchEvent(new CustomEvent("input"));
  };

  // Handle input event for suggestions
  input.oninput = async (event) => {
    const value = event.target.value.trim();
    if (!value) return;

    // Fetch suggestions
    const results = await fetchAutoSuggest(value);

    // Clear existing suggestions
    datalist.innerHTML = "";

    // Populate datalist with new suggestions
    results.forEach((result) => {
      const formattedResult = formatFn(result);
      const option = document.createElement("option");
      option.value = formattedResult; // Full formatFn output
      datalist.appendChild(option);
    });
  };

  // Clear input on focus
  input.onfocus = () => {
    input.value = ""; // Clear the input so the user can type a new search
  };

  return form;
}



```

```js
// Import external dependencies
import moment from 'npm:moment'
import { es_ES, etiquetasTrimestres } from "./components/config.js";

// Set the locale for D3 formatting
d3.formatDefaultLocale(es_ES);
```