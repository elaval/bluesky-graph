---
toc: false
sql:
  master: https://raw.githubusercontent.com/elaval/BskyFollowersTrend/refs/heads/main/master.parquet
  followers_log: https://raw.githubusercontent.com/elaval/BskyFollowersTrend/refs/heads/main/followers_log.parquet
---



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

## Número de seguidores en Bluesky
### Línea de tiempo

*Ultima actualización: ${moment.utc(_.chain([...master]).map(d => d.timestamp).max().value()).format("D MMM YYYY, HH:mm")}*

¿Quieres saber cómo ha evolucionado el número de tus seguidores en el tiempo?

Desde el 4 de Diciembre de 2024 un proceso automático monitorea 4 veces al día el número de seguidores que tiene un conjunto de cuentas en Bluesky y almacena los datos.  Esta página muestra la evolución en el tiempo del número de seguidores para alguna de las cuentas monitoreadas.  

Las cuentas monitoreadas, por ahora, son aquellas que siguen [@elaval.bsky.social](https://bsky.app/profile/elaval.bsky.social) y el monitoreo se inició el 4 de Diciembre de 2024 (o el día que empezaron a seguir a @elaval.bsky.social si es posterior a esa fecha).  

Este es una exploración para analizar el comportamiento de seguidores en Bluesky y probar maneras de presentar y compartir la información.  En el futuro pueden haber moficicaciones a la frecuencia de monitoreo o a los criterios para configurar la lista de cuentas a monitorear (por ahora es una manera simple de partir)


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
  const initialData = selectedUser && _.concat(
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
    subtitle: `La línea roja punteada es una proyección desde el momento de creación de la cuenta hasta el inicio del monitoreo.`,
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
// This function creates a simple autocomplete widget.
// It returns a container element which Observable can treat as an input view.
// When a user selects an item from the dropdown, container.value is updated and an input event is dispatched.
function createAutocomplete_(items = []) {
  const container = document.createElement("div");
  container.style.position = "relative";
  container.style.width = "250px";
  container.style.fontFamily = "sans-serif";
  container.value = ""; // Initialize for Observable

  // Create text input
  const input = document.createElement("input");
  input.type = "text";
  input.style.width = "100%";
  input.style.boxSizing = "border-box";
  input.placeholder = "Type to search...";
  input.style.padding = "8px";
  input.style.border = "1px solid #ccc";
  input.style.borderRadius = "4px";
  container.appendChild(input);

  // Create dropdown container
  const dropdown = document.createElement("div");
  dropdown.style.position = "absolute";
  dropdown.style.width = "100%";
  dropdown.style.boxSizing = "border-box";
  dropdown.style.border = "1px solid #ccc";
  dropdown.style.borderTop = "none";
  dropdown.style.background = "#fff";
  dropdown.style.zIndex = "999";
  dropdown.style.display = "none";
  dropdown.style.maxHeight = "150px";
  dropdown.style.overflowY = "auto";
  dropdown.style.borderRadius = "0 0 4px 4px";
  container.appendChild(dropdown);

  function clearDropdown() {
    dropdown.innerHTML = "";
    dropdown.style.display = "none";
  }

  function updateDropdown(value) {
    clearDropdown();
    if (!value) return;

    const filtered = items
      .filter(item => item.toLowerCase().includes(value.toLowerCase()))
      .slice(0, 10);

    if (filtered.length > 0) {
      filtered.forEach(item => {
        const option = document.createElement("div");
        option.style.padding = "8px";
        option.style.cursor = "pointer";
        option.style.borderBottom = "1px solid #eee";
        option.textContent = item;

        option.addEventListener("mousedown", () => {
          input.value = item;
          container.value = item; // Update container's value
          container.dispatchEvent(new Event("input")); // Dispatch event for Observable
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

  // Input event handlers
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



```

```js
// Import external dependencies
import moment from 'npm:moment'
import { es_ES, etiquetasTrimestres } from "./components/config.js";

// Set the locale for D3 formatting
d3.formatDefaultLocale(es_ES);
```