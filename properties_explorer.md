

```dataviewjs
/* ============================================================
   EXPLORATEUR DE NOTES - DATAVIEWJS
   ============================================================ */

const root = dv.container;

/* ============================================================
   ÉTAT
   ============================================================ */

const state = {
    selectedFolder: "",
    selectedImageProperty: "",
    search: "",
    sortField: "name",
    sortDirection: "asc",
    valueSort: "count",
    criteria: [],
    statisticsOpen: false
};


/* ============================================================
   FICHIERS
   ============================================================ */

const allFiles = app.vault
    .getMarkdownFiles()
    .sort((a, b) =>
        a.path.localeCompare(b.path, "fr")
    );

const frontmatterCache = new Map();

for (const file of allFiles) {
    const cache =
        app.metadataCache.getFileCache(file);

    frontmatterCache.set(
        file.path,
        cache?.frontmatter || {}
    );
}


/* ============================================================
   OUTILS
   ============================================================ */

function escapeHtml(value) {
    return String(value ?? "")
        .replace(/&/g, "&amp;")
        .replace(/</g, "&lt;")
        .replace(/>/g, "&gt;")
        .replace(/"/g, "&quot;")
        .replace(/'/g, "&#039;");
}

function normalize(value) {
    return String(value ?? "")
        .trim()
        .toLowerCase();
}

function displayValue(value) {

    if (value === null || value === undefined) {
        return "";
    }

    if (Array.isArray(value)) {
        return value
            .flatMap(v => flattenValue(v))
            .map(v => displayValue(v))
            .join(", ");
    }

    if (typeof value === "object") {

        if (value.path !== undefined) {
            return String(value.path);
        }

        return JSON.stringify(value);
    }

    return String(value);
}

function flattenValue(value) {

    if (value === null || value === undefined) {
        return [];
    }

    if (Array.isArray(value)) {
        return value.flatMap(v =>
            flattenValue(v)
        );
    }

    if (
        typeof value === "object" &&
        value.path !== undefined
    ) {
        return [value.path];
    }

    return [value];
}


/* ============================================================
   FRONTMATTER
   ============================================================ */

function getFrontmatter(file) {
    return frontmatterCache.get(file.path) || {};
}

function getPropertyValues(file, property) {

    if (property === "Date de création du fichier") {
        return [new Date(file.stat.ctime).toISOString()];
    }

    if (property === "date de modification du fichier") {
        return [new Date(file.stat.mtime).toISOString()];
    }

    if (!property) {
        return [];
    }

    const fm = getFrontmatter(file);

    if (!(property in fm)) {
        return [];
    }

    return flattenValue(fm[property]);
}

function hasPropertyValue(file, property) {

    const values =
        getPropertyValues(file, property);

    return values.some(value =>
        String(value ?? "").trim() !== ""
    );
}


/* ============================================================
   DOSSIERS
   ============================================================ */

function getFolders() {

    const folders = new Set([""]);

    for (const file of allFiles) {

        const parts =
            file.path.split("/");

        parts.pop();

        let current = "";

        for (const part of parts) {

            current = current
                ? `${current}/${part}`
                : part;

            folders.add(current);
        }
    }

    return Array.from(folders)
        .sort((a, b) =>
            a.localeCompare(b, "fr")
        );
}

function getFilesForFolder(folder) {

    if (!folder) {
        return allFiles;
    }

    const prefix =
        folder.endsWith("/")
            ? folder
            : folder + "/";

    return allFiles.filter(file =>
        file.path.startsWith(prefix)
    );
}


/* ============================================================
   PROPRIÉTÉS
   ============================================================ */

function getProperties(files = allFiles) {

    const properties = new Set();

    for (const file of files) {

        const fm =
            getFrontmatter(file);

        for (const key of Object.keys(fm)) {
            properties.add(key);
        }
    }

    properties.add("Date de création du fichier");
    properties.add("date de modification du fichier");

    return Array.from(properties)
        .sort((a, b) =>
            a.localeCompare(b, "fr")
        );
}


/* ============================================================
   RECHERCHE
   ============================================================ */

function noteMatchesSearch(file) {

    if (!state.search.trim()) {
        return true;
    }

    const query =
        normalize(state.search);

    const fm =
        getFrontmatter(file);

    const frontmatterText =
        Object.entries(fm)
            .map(([key, value]) =>
                `${key} ${displayValue(value)}`
            )
            .join(" ");

    const text =
        `${file.basename} ${file.path} ${frontmatterText}`
            .toLowerCase();

    return text.includes(query);
}


/* ============================================================
   CRITÈRES
   ============================================================ */

function createCriterion() {

    return {
        property: "",
        type: "text",

        withValues: [],
        withoutValues: [],

        requireAll: false,

        min: null,
        max: null,

        isNull: false,
        isNotNull: false
    };
}


/* ============================================================
   DÉTECTION TYPE PROPRIÉTÉ
   ============================================================ */

function detectPropertyType(
    property,
    files
) {

    const values = [];

    for (const file of files) {

        for (
            const value
            of getPropertyValues(
                file,
                property
            )
        ) {

            if (
                value !== null &&
                value !== undefined &&
                String(value).trim() !== ""
            ) {
                values.push(value);
            }
        }
    }

    if (!values.length) {
        return "text";
    }

    const date = values.length > 0 && values.every(value => {
        const text = String(value).trim();
        return /^\d{4}-\d{2}-\d{2}(?:[T\s].*)?$/.test(text)
            && !Number.isNaN(Date.parse(text));
    });

    if (date) return "date";

    const numeric =
        values.every(value => {

            if (
                typeof value === "number"
            ) {
                return true;
            }

            const text =
                String(value).trim();

            return (
                text !== "" &&
                !Number.isNaN(
                    Number(text)
                )
            );
        });

    return numeric
        ? "number"
        : "text";
}


/* ============================================================
   CRITÈRE
   ============================================================ */

function noteMatchesCriterion(
    file,
    criterion
) {

    if (!criterion.property) {
        return true;
    }

    const values =
        getPropertyValues(
            file,
            criterion.property
        );

    const nonEmptyValues =
        values.filter(value =>
            String(value ?? "").trim() !== ""
        );

    const isNull =
        nonEmptyValues.length === 0;

    /* --------------------------------------------------------
       VALEUR NULLE
       -------------------------------------------------------- */

    if (criterion.isNull) {
        return isNull;
    }

    /* --------------------------------------------------------
       VALEUR NON NULLE
       -------------------------------------------------------- */

    if (criterion.isNotNull) {
        return !isNull;
    }

    /*
       Si la propriété est vide et qu'on cherche
       une valeur réelle, elle ne correspond pas.
    */

    if (isNull) {
        return false;
    }


    /* ========================================================
       DATE
       ======================================================== */

    if (criterion.type === "date") {
        const timestamps = nonEmptyValues
            .map(value => Date.parse(String(value)))
            .filter(value => !Number.isNaN(value));

        if (!timestamps.length) return false;

        const min = criterion.min === null || criterion.min === ""
            ? -Infinity : Date.parse(`${criterion.min}T00:00:00`);
        const max = criterion.max === null || criterion.max === ""
            ? Infinity : Date.parse(`${criterion.max}T23:59:59.999`);

        return timestamps.some(timestamp => timestamp >= min && timestamp <= max);
    }

    /* ========================================================
       NUMÉRIQUE
       ========================================================
       */

    if (criterion.type === "number") {

        const numbers =
            nonEmptyValues
                .map(value =>
                    Number(value)
                )
                .filter(value =>
                    !Number.isNaN(value)
                );

        if (!numbers.length) {
            return false;
        }

        const min =
            criterion.min === null ||
            criterion.min === ""
                ? -Infinity
                : Number(criterion.min);

        const max =
            criterion.max === null ||
            criterion.max === ""
                ? Infinity
                : Number(criterion.max);

        /*
           Pour une propriété contenant plusieurs
           nombres, au moins un nombre doit être
           dans l'intervalle.
        */

        return numbers.some(number =>
            number >= min &&
            number <= max
        );
    }


    /* ========================================================
       TEXTE
       ======================================================== */

    const normalizedValues =
        nonEmptyValues.map(value =>
            normalize(
                displayValue(value)
            )
        );

    const withSelected =
        (criterion.withValues || [])
            .map(value =>
                normalize(value)
            );

    const withoutSelected =
        (criterion.withoutValues || [])
            .map(value =>
                normalize(value)
            );


    /* --------------------------------------------------------
       AVEC
       -------------------------------------------------------- */

    if (withSelected.length > 0) {

        let matches;

        if (criterion.requireAll) {

            matches =
                withSelected.every(
                    wanted =>
                        normalizedValues
                            .includes(wanted)
                );

        } else {

            matches =
                withSelected.some(
                    wanted =>
                        normalizedValues
                            .includes(wanted)
                );
        }

        if (!matches) {
            return false;
        }
    }


    /* --------------------------------------------------------
       SANS
       -------------------------------------------------------- */

    if (withoutSelected.length > 0) {

        const forbidden =
            withoutSelected.some(
                wanted =>
                    normalizedValues
                        .includes(wanted)
            );

        if (forbidden) {
            return false;
        }
    }

    return true;
}


/* ============================================================
   FILTRAGE GLOBAL
   ============================================================ */

function fileMatchesAllCriteria(
    file,
    ignoredCriterionIndex = null
) {

    if (!noteMatchesSearch(file)) {
        return false;
    }

    for (
        let i = 0;
        i < state.criteria.length;
        i++
    ) {

        if (
            i === ignoredCriterionIndex
        ) {
            continue;
        }

        if (
            !noteMatchesCriterion(
                file,
                state.criteria[i]
            )
        ) {
            return false;
        }
    }

    return true;
}

function getFilteredFiles() {

    let files =
        getFilesForFolder(
            state.selectedFolder
        );

    return files.filter(file =>
        fileMatchesAllCriteria(file)
    );
}


/* ============================================================
   VALEURS DISPONIBLES
   ============================================================ */

function getAvailableValuesForCriterion(
    criterionIndex,
    property,
    valuesKey = "withValues"
) {

    if (!property) {
        return [];
    }

    const counts = new Map();

    let files = getFilesForFolder(
        state.selectedFolder
    );

    files = files.filter(file =>
        noteMatchesSearch(file)
    );

    const currentCriterion =
        state.criteria[criterionIndex];

    /*
       Appliquer tous les critères autres que celui
       dont on construit actuellement la liste.
    */
    for (
        let i = 0;
        i < state.criteria.length;
        i++
    ) {

        if (i === criterionIndex) {
            continue;
        }

        files = files.filter(file =>
            noteMatchesCriterion(
                file,
                state.criteria[i]
            )
        );
    }

    /*
       Si "Toutes les valeurs AVEC doivent être présentes"
       est activé, on applique les valeurs Avec déjà choisies.

       Cela permet de recalculer les valeurs restantes dans
       Avec ET dans Sans à partir des notes compatibles.
    */
    if (
        currentCriterion &&
        currentCriterion.requireAll &&
        currentCriterion.withValues &&
        currentCriterion.withValues.length > 0
    ) {

        const valuesToRequire =
            currentCriterion.withValues.map(value =>
                normalize(value)
            );

        files = files.filter(file => {

            const fileValues = new Set(
                getPropertyValues(
                    file,
                    property
                ).map(value =>
                    normalize(displayValue(value))
                )
            );

            return valuesToRequire.every(value =>
                fileValues.has(value)
            );
        });
    }

    for (const file of files) {

        const values = getPropertyValues(
            file,
            property
        );

        const uniqueValues = new Set(
            values
                .map(value =>
                    displayValue(value)
                )
                .filter(value =>
                    value.trim() !== ""
                )
        );

        for (const value of uniqueValues) {

            counts.set(
                value,
                (counts.get(value) || 0) + 1
            );
        }
    }

    let result = Array.from(
        counts.entries()
    );

    if (state.valueSort === "count") {

        result.sort(
            (a, b) =>
                b[1] - a[1] ||
                a[0].localeCompare(
                    b[0],
                    "fr"
                )
        );

    } else {

        result.sort(
            (a, b) =>
                a[0].localeCompare(
                    b[0],
                    "fr"
                )
        );
    }

    return result;
}

/* ============================================================
   IMAGES
   ============================================================ */

function cleanImageReference(value) {

    let result =
        String(value ?? "").trim();

    /*
       ![[image.png]]
    */

    result =
        result.replace(
            /^!\[\[/,
            ""
        );

    /*
       [[image.png]]
    */

    result =
        result.replace(
            /^\[\[/,
            ""
        );

    /*
       ]] final
    */

    result =
        result.replace(
            /\]\]$/,
            ""
        );

    /*
       Alias Obsidian :
       [[image.png|Nom]]
    */

    if (result.includes("|")) {

        result =
            result.split("|")[0];
    }

    /*
       Dimensions éventuelles
    */

    result =
        result.replace(
            /#\d+(x\d+)?$/,
            ""
        );

    return result.trim();
}


function getImagePaths(
    file,
    property
) {

    if (!property) {
        return [];
    }

    const rawValues =
        getPropertyValues(
            file,
            property
        );

    const results = [];

    for (const rawValue of rawValues) {

        const cleaned =
            cleanImageReference(
                rawValue
            );

        if (!cleaned) {
            continue;
        }

        /*
           URL externe
        */

        if (
            /^https?:\/\//i.test(
                cleaned
            )
        ) {

            results.push(cleaned);
            continue;
        }

        /*
           Fichier image du vault
        */

        const pathWithoutQuery =
            cleaned.split("?")[0];

        if (
            /\.(png|jpe?g|gif|webp|svg|avif|bmp|tiff?)$/i
                .test(pathWithoutQuery)
        ) {

            results.push(cleaned);
        }
    }

    return results;
}


function getImageUrl(
    imagePath,
    sourceFile
) {

    const cleaned =
        cleanImageReference(
            imagePath
        );

    if (!cleaned) {
        return null;
    }

    /*
       URL externe
    */

    if (
        /^https?:\/\//i.test(
            cleaned
        )
    ) {
        return cleaned;
    }

    /*
       Résolution par Obsidian
    */

    const linkedFile =
        app.metadataCache
            .getFirstLinkpathDest(
                cleaned,
                sourceFile.path
            );

    if (linkedFile) {

        return app.vault
            .getResourcePath(
                linkedFile
            );
    }

    /*
       Tentative directe
    */

    const directFile =
        app.vault
            .getAbstractFileByPath(
                cleaned
            );

    if (
        directFile &&
        directFile.path
    ) {

        return app.vault
            .getResourcePath(
                directFile
            );
    }

    return null;
}


function renderNoteImage(
    file,
    container
) {

    if (
        !state.selectedImageProperty
    ) {
        return;
    }

    const imagePaths =
        getImagePaths(
            file,
            state.selectedImageProperty
        );

    for (
        const imagePath
        of imagePaths
    ) {

        const url =
            getImageUrl(
                imagePath,
                file
            );

        if (!url) {
            continue;
        }

        const img =
            document.createElement("img");

        img.className =
            "explorer-result-image";

        img.src = url;

        img.alt =
            file.basename;

        img.loading =
            "lazy";

        img.onerror = () => {
            img.remove();
        };

        container.appendChild(
            img
        );

        return;
    }
}


/* ============================================================
   STATISTIQUES
   ============================================================ */

function getStatisticFiles(
    property
) {

    let files =
        getFilesForFolder(
            state.selectedFolder
        );

    files =
        files.filter(file =>
            noteMatchesSearch(file)
        );

    /*
       On ignore les critères portant
       sur la propriété dont on calcule
       les statistiques.
    */

    for (
        let i = 0;
        i < state.criteria.length;
        i++
    ) {

        const criterion =
            state.criteria[i];

        if (
            criterion.property ===
            property
        ) {
            continue;
        }

        files =
            files.filter(file =>
                noteMatchesCriterion(
                    file,
                    criterion
                )
            );
    }

    return files;
}


function calculateStatistics(
    property
) {

    const files =
        getStatisticFiles(
            property
        );

    const nullFiles = [];
    const nonNullFiles = [];

    /*
       valeur -> Set de fichiers
    */

    const valueFiles =
        new Map();

    for (const file of files) {

        const values =
            getPropertyValues(
                file,
                property
            )
            .map(value =>
                displayValue(value)
            )
            .filter(value =>
                value.trim() !== ""
            );

        const uniqueValues =
            Array.from(
                new Set(values)
            );

        if (
            uniqueValues.length === 0
        ) {

            nullFiles.push(file);

        } else {

            nonNullFiles.push(file);
        }

        for (
            const value
            of uniqueValues
        ) {

            if (
                !valueFiles.has(value)
            ) {

                valueFiles.set(
                    value,
                    new Set()
                );
            }

            valueFiles
                .get(value)
                .add(file);
        }
    }

    const distinctValues =
        Array.from(
            valueFiles.entries()
        )
        .map(
            ([value, filesSet]) => ({
                value,
                count: filesSet.size,
                files:
                    Array.from(
                        filesSet
                    )
            })
        );

    distinctValues.sort(
        (a, b) =>
            b.count - a.count ||
            a.value.localeCompare(
                b.value,
                "fr"
            )
    );

    return {
        files,
        nullFiles,
        nonNullFiles,
        distinctValues
    };
}


/* ============================================================
   LISTE DE NOTES DES STATISTIQUES
   ============================================================ */

function renderStatisticNotes(
    container,
    files,
    property
) {

    container.innerHTML = "";

    if (!files.length) {

        const empty =
            document.createElement("div");

        empty.className =
            "explorer-stat-empty";

        empty.textContent =
            "Aucune note.";

        container.appendChild(
            empty
        );

        return;
    }

    for (const file of files) {

        const row =
            document.createElement("div");

        row.className =
            "explorer-stat-note";

        const name =
            document.createElement("div");

        name.className =
            "explorer-stat-note-name";

        const link =
            document.createElement("a");

        link.href = "#";

        link.textContent =
            file.basename;

        link.onclick =
            async event => {

                event.preventDefault();

                await app.workspace
                    .getLeaf(false)
                    .openFile(file);
            };

        name.appendChild(
            link
        );

        const path =
            document.createElement("div");

        path.className =
            "explorer-stat-note-path";

        path.textContent =
            file.path;

        row.appendChild(
            name
        );

        row.appendChild(
            path
        );

        if (property) {

            const values =
                getPropertyValues(
                    file,
                    property
                );

            if (values.length) {

                const valuesDiv =
                    document.createElement(
                        "div"
                    );

                valuesDiv.className =
                    "explorer-stat-note-values";

                valuesDiv.textContent =
                    values
                        .map(displayValue)
                        .join(", ");

                row.appendChild(
                    valuesDiv
                );
            }
        }

        container.appendChild(
            row
        );
    }
}


/* ============================================================
   VALEURS DISTINCTES
   ============================================================ */

function renderDistinctValues(
    container,
    values,
    property
) {

    container.innerHTML = "";

    if (!values.length) {

        const empty =
            document.createElement("div");

        empty.className =
            "explorer-stat-empty";

        empty.textContent =
            "Aucune valeur.";

        container.appendChild(
            empty
        );

        return;
    }

    for (const item of values) {

        const wrapper =
            document.createElement("div");

        wrapper.className =
            "explorer-stat-distinct";

        const button =
            document.createElement("button");

        button.className =
            "explorer-stat-value";

        button.textContent =
            `${item.value} (${item.count})`;

        const details =
            document.createElement("div");

        details.className =
            "explorer-stat-details";

        details.style.display =
            "none";

        button.onclick = () => {

            const visible =
                details.style.display !==
                "none";

            details.style.display =
                visible
                    ? "none"
                    : "block";

            if (!visible) {

                renderStatisticNotes(
                    details,
                    item.files,
                    property
                );
            }
        };

        wrapper.appendChild(
            button
        );

        wrapper.appendChild(
            details
        );

        container.appendChild(
            wrapper
        );
    }
}


/* ============================================================
   AFFICHAGE STATISTIQUES
   ============================================================ */

function renderStatistics(
    container,
    files
) {

    container.innerHTML = "";

    const properties =
        getProperties(files);

    if (!properties.length) {

        const empty =
            document.createElement("div");

        empty.className =
            "explorer-stat-empty";

        empty.textContent =
            "Aucune propriété.";

        container.appendChild(
            empty
        );

        return;
    }

    for (
        const property
        of properties
    ) {

        const stats =
            calculateStatistics(
                property
            );

        const block =
            document.createElement("div");

        block.className =
            "explorer-stat-criterion";

        const title =
            document.createElement("div");

        title.className =
            "explorer-stat-criterion-title";

        title.textContent =
            property;

        block.appendChild(
            title
        );

        const lines =
            document.createElement("div");

        lines.className =
            "explorer-stat-lines";


        /* VALEURS NULLES */

        const nullButton =
            document.createElement(
                "button"
            );

        nullButton.className =
            "explorer-stat-button";

        nullButton.textContent =
            `Valeurs nulles (${stats.nullFiles.length})`;

        const nullDetails =
            document.createElement(
                "div"
            );

        nullDetails.className =
            "explorer-stat-details";

        nullDetails.style.display =
            "none";

        nullButton.onclick = () => {

            const visible =
                nullDetails.style.display !==
                "none";

            nullDetails.style.display =
                visible
                    ? "none"
                    : "block";

            if (!visible) {

                renderStatisticNotes(
                    nullDetails,
                    stats.nullFiles,
                    property
                );
            }
        };

        lines.appendChild(
            nullButton
        );

        lines.appendChild(
            nullDetails
        );


        /* VALEURS NON NULLES */

        const nonNullButton =
            document.createElement(
                "button"
            );

        nonNullButton.className =
            "explorer-stat-button";

        nonNullButton.textContent =
            `Valeurs non nulles (${stats.nonNullFiles.length})`;

        const nonNullDetails =
            document.createElement(
                "div"
            );

        nonNullDetails.className =
            "explorer-stat-details";

        nonNullDetails.style.display =
            "none";

        nonNullButton.onclick = () => {

            const visible =
                nonNullDetails.style.display !==
                "none";

            nonNullDetails.style.display =
                visible
                    ? "none"
                    : "block";

            if (!visible) {

                renderStatisticNotes(
                    nonNullDetails,
                    stats.nonNullFiles,
                    property
                );
            }
        };

        lines.appendChild(
            nonNullButton
        );

        lines.appendChild(
            nonNullDetails
        );


        /* VALEURS DISTINCTES */

        const distinctButton =
            document.createElement(
                "button"
            );

        distinctButton.className =
            "explorer-stat-button";

        distinctButton.textContent =
            `Valeurs distinctes (${stats.distinctValues.length})`;

        const distinctDetails =
            document.createElement(
                "div"
            );

        distinctDetails.className =
            "explorer-stat-details";

        distinctDetails.style.display =
            "none";

        distinctButton.onclick = () => {

            const visible =
                distinctDetails.style.display !==
                "none";

            distinctDetails.style.display =
                visible
                    ? "none"
                    : "block";

            if (!visible) {

                renderDistinctValues(
                    distinctDetails,
                    stats.distinctValues,
                    property
                );
            }
        };

        lines.appendChild(
            distinctButton
        );

        lines.appendChild(
            distinctDetails
        );

        block.appendChild(
            lines
        );

        container.appendChild(
            block
        );
    }
}


/* ============================================================
   AUTOCOMPLÉTION
   ============================================================ */

function renderSelectedValues(
    criterion,
    target,
    valuesKey
) {

    target.innerHTML = "";

    const values =
        criterion[valuesKey] || [];

    for (const value of values) {

        const chip =
            document.createElement(
                "span"
            );

        chip.className =
            "explorer-selected-value";

        chip.textContent =
            value;

        const remove =
            document.createElement(
                "button"
            );

        remove.type =
            "button";

        remove.textContent =
            "×";

        remove.className =
            "explorer-selected-value-remove";

        remove.onclick = () => {

            criterion[valuesKey] =
                criterion[valuesKey]
                    .filter(
                        v => v !== value
                    );

            renderAll();
        };

        chip.appendChild(
            remove
        );

        target.appendChild(
            chip
        );
    }
}


function renderAutocomplete(
    criterionIndex,
    criterion,
    property,
    valuesKey,
    container
) {

    container.innerHTML = "";

    if (!property) {
        return;
    }

    const wrapper =
        document.createElement(
            "div"
        );

    wrapper.className =
        "explorer-autocomplete";

    const input =
        document.createElement(
            "input"
        );

    input.type =
        "text";

    input.className =
        "explorer-autocomplete-input";

    input.placeholder =
        "Rechercher une valeur...";

    const selected =
        document.createElement(
            "div"
        );

    selected.className =
        "explorer-selected-values";

    renderSelectedValues(
        criterion,
        selected,
        valuesKey
    );

    const list =
        document.createElement(
            "div"
        );

    list.className =
        "explorer-autocomplete-list";

    wrapper.appendChild(
        input
    );

    wrapper.appendChild(
        selected
    );

    wrapper.appendChild(
        list
    );

    container.appendChild(
        wrapper
    );


    function refreshList() {

        list.innerHTML = "";

        const query =
            normalize(
                input.value
            );

        const available =
            getAvailableValuesForCriterion(
                criterionIndex,
                property,
                valuesKey
            );

        const selectedValues =
            new Set(
                criterion[valuesKey] || []
            );

        const filtered =
            available.filter(
                ([value]) =>
                    !selectedValues.has(value) &&
                    (
                        !query ||
                        normalize(value)
                            .includes(query)
                    )
            );

        for (
            const [value, count]
            of filtered.slice(0, 100)
        ) {

            const item =
                document.createElement(
                    "div"
                );

            item.className =
                "explorer-autocomplete-item";

            const text =
                document.createElement(
                    "span"
                );

            text.textContent =
                value;

            const countSpan =
                document.createElement(
                    "span"
                );

            countSpan.className =
                "explorer-autocomplete-count";

            countSpan.textContent =
                count;

            item.appendChild(
                text
            );

            item.appendChild(
                countSpan
            );

            item.onclick = () => {

                if (
                    !criterion[valuesKey]
                ) {
                    criterion[valuesKey] =
                        [];
                }

                criterion[valuesKey].push(
                    value
                );

                renderAll();
            };

            list.appendChild(
                item
            );
        }
    }

    input.addEventListener(
        "input",
        refreshList
    );

    input.addEventListener(
        "focus",
        refreshList
    );
}


/* ============================================================
   CRITÈRES - AFFICHAGE
   ============================================================ */

function renderCriteria(container) {

    container.innerHTML = "";

    if (!state.criteria.length) {

        const empty =
            document.createElement(
                "div"
            );

        empty.className =
            "explorer-empty";

        empty.textContent =
            "Aucun critère.";

        container.appendChild(
            empty
        );

        return;
    }

    state.criteria.forEach(
        (criterion, index) => {

            const block =
                document.createElement(
                    "div"
                );

            block.className =
                "explorer-criterion";

            const header =
                document.createElement(
                    "div"
                );

            header.className =
                "explorer-criterion-header";

            const title =
                document.createElement(
                    "strong"
                );

            title.textContent =
                `Critère ${index + 1}`;

            header.appendChild(
                title
            );

            const remove =
                document.createElement(
                    "button"
                );

            remove.textContent =
                "Supprimer";

            remove.onclick = () => {

                state.criteria.splice(
                    index,
                    1
                );

                renderAll();
            };

            header.appendChild(
                remove
            );

            block.appendChild(
                header
            );


            const body =
                document.createElement(
                    "div"
                );

            body.className =
                "explorer-criterion-body";


            /* PROPRIÉTÉ */

            const propertyLabel =
                document.createElement(
                    "label"
                );

            propertyLabel.textContent =
                "Propriété";

            const propertySelect =
                document.createElement(
                    "select"
                );

            const emptyOption =
                document.createElement(
                    "option"
                );

            emptyOption.value =
                "";

            emptyOption.textContent =
                "Choisir une propriété";

            propertySelect.appendChild(
                emptyOption
            );

            const properties =
                getProperties(
                    getFilesForFolder(
                        state.selectedFolder
                    )
                );

            for (
                const property
                of properties
            ) {

                const option =
                    document.createElement(
                        "option"
                    );

                option.value =
                    property;

                option.textContent =
                    property;

                option.selected =
                    criterion.property ===
                    property;

                propertySelect.appendChild(
                    option
                );
            }

            propertySelect.onchange =
                () => {

                    criterion.property =
                        propertySelect.value;

                    criterion.type =
                        detectPropertyType(
                            criterion.property,
                            getFilesForFolder(
                                state.selectedFolder
                            )
                        );

                    criterion.withValues =
                        [];

                    criterion.withoutValues =
                        [];

                    criterion.min =
                        null;

                    criterion.max =
                        null;

                    criterion.isNull =
                        false;

                    criterion.isNotNull =
                        false;

                    renderAll();
                };


            body.appendChild(
                propertyLabel
            );

            body.appendChild(
                propertySelect
            );


            /* =================================================
               NULL / NON NULL
               ================================================= */

            if (criterion.property) {

                const nullContainer =
                    document.createElement(
                        "div"
                    );

                nullContainer.className =
                    "explorer-null-options";


                /* NULLE */

                const nullLabel =
                    document.createElement(
                        "label"
                    );

                nullLabel.className =
                    "explorer-check-label";

                const nullCheckbox =
                    document.createElement(
                        "input"
                    );

                nullCheckbox.type =
                    "checkbox";

                nullCheckbox.checked =
                    criterion.isNull;

                nullCheckbox.onchange =
                    () => {

                        criterion.isNull =
                            nullCheckbox.checked;

                        if (
                            criterion.isNull
                        ) {

                            criterion.isNotNull =
                                false;
                        }

                        renderAll();
                    };

                const nullText =
                    document.createElement(
                        "span"
                    );

                nullText.textContent =
                    "Valeur nulle uniquement";

                nullLabel.appendChild(
                    nullCheckbox
                );

                nullLabel.appendChild(
                    nullText
                );


                /* NON NULLE */

                const notNullLabel =
                    document.createElement(
                        "label"
                    );

                notNullLabel.className =
                    "explorer-check-label";

                const notNullCheckbox =
                    document.createElement(
                        "input"
                    );

                notNullCheckbox.type =
                    "checkbox";

                notNullCheckbox.checked =
                    criterion.isNotNull;

                notNullCheckbox.onchange =
                    () => {

                        criterion.isNotNull =
                            notNullCheckbox.checked;

                        if (
                            criterion.isNotNull
                        ) {

                            criterion.isNull =
                                false;
                        }

                        renderAll();
                    };

                const notNullText =
                    document.createElement(
                        "span"
                    );

                notNullText.textContent =
                    "Valeur non nulle uniquement";

                notNullLabel.appendChild(
                    notNullCheckbox
                );

                notNullLabel.appendChild(
                    notNullText
                );

                nullContainer.appendChild(
                    nullLabel
                );

                nullContainer.appendChild(
                    notNullLabel
                );

                body.appendChild(
                    nullContainer
                );


                /*
                   Les critères Avec/Sans ou le
                   numérique ne sont utiles que
                   si on ne force pas null/non-null.
                */

                if (
                    !criterion.isNull &&
                    !criterion.isNotNull
                ) {

                    if (criterion.type === "number") {
                        renderNumericCriterion(body, criterion, index);
                    } else if (criterion.type === "date") {
                        renderDateCriterion(body, criterion);
                    } else {
                        renderTextCriterion(body, index, criterion);
                    }
                }
            }


            block.appendChild(
                body
            );

            container.appendChild(
                block
            );
        }
    );
}


/* ============================================================
   CRITÈRE TEXTE
   ============================================================ */

function renderTextCriterion(
    container,
    criterionIndex,
    criterion
) {

    const forms =
        document.createElement(
            "div"
        );

    forms.className =
        "explorer-text-forms";


    /* AVEC */

    const withForm =
        document.createElement(
            "div"
        );

    withForm.className =
        "explorer-value-form";

    const withTitle =
        document.createElement(
            "div"
        );

    withTitle.className =
        "explorer-value-form-title";

    withTitle.textContent =
        "Avec";

    withForm.appendChild(
        withTitle
    );

    const withAutocomplete =
        document.createElement(
            "div"
        );

    renderAutocomplete(
        criterionIndex,
        criterion,
        criterion.property,
        "withValues",
        withAutocomplete
    );

    withForm.appendChild(
        withAutocomplete
    );


    /* TOUTES LES VALEURS */

    const allWrapper =
        document.createElement(
            "label"
        );

    allWrapper.className =
        "explorer-with-all";

    const checkbox =
        document.createElement(
            "input"
        );

    checkbox.type =
        "checkbox";

    checkbox.checked =
        criterion.requireAll;

    checkbox.onchange = () => {

        criterion.requireAll =
            checkbox.checked;

        renderAll();
    };

    const text =
        document.createElement(
            "span"
        );

    text.textContent =
        "Toutes les valeurs AVEC doivent être présentes";

    allWrapper.appendChild(
        checkbox
    );

    allWrapper.appendChild(
        text
    );

    withForm.appendChild(
        allWrapper
    );


    /* SANS */

    const withoutForm =
        document.createElement(
            "div"
        );

    withoutForm.className =
        "explorer-value-form";

    const withoutTitle =
        document.createElement(
            "div"
        );

    withoutTitle.className =
        "explorer-value-form-title";

    withoutTitle.textContent =
        "Sans";

    withoutForm.appendChild(
        withoutTitle
    );

    const withoutAutocomplete =
        document.createElement(
            "div"
        );

    renderAutocomplete(
        criterionIndex,
        criterion,
        criterion.property,
        "withoutValues",
        withoutAutocomplete
    );

    withoutForm.appendChild(
        withoutAutocomplete
    );


    forms.appendChild(
        withForm
    );

    forms.appendChild(
        withoutForm
    );

    container.appendChild(
        forms
    );
}


/* ============================================================
   CRITÈRE DATE - DOUBLE DATE PICKER
   ============================================================ */

function renderDateCriterion(container, criterion) {
    const wrapper = document.createElement("div");
    wrapper.className = "explorer-date-forms";

    const fromLabel = document.createElement("label");
    fromLabel.textContent = "Date de début";
    const from = document.createElement("input");
    from.type = "date";
    from.value = criterion.min || "";
    from.onchange = () => { criterion.min = from.value || null; renderAll(); };
    fromLabel.appendChild(from);

    const toLabel = document.createElement("label");
    toLabel.textContent = "Date de fin";
    const to = document.createElement("input");
    to.type = "date";
    to.value = criterion.max || "";
    to.onchange = () => { criterion.max = to.value || null; renderAll(); };
    toLabel.appendChild(to);

    wrapper.appendChild(fromLabel);
    wrapper.appendChild(toLabel);
    container.appendChild(wrapper);
}


/* ============================================================
   CRITÈRE NUMÉRIQUE AVEC SLIDER
   ============================================================ */

function renderNumericCriterion(
    container,
    criterion,
    criterionIndex = null
) {

    let files =
        getFilesForFolder(
            state.selectedFolder
        );

    files = files.filter(file =>
        noteMatchesSearch(file)
    );

    for (let i = 0; i < state.criteria.length; i++) {
        if (i === criterionIndex) continue;
        files = files.filter(file =>
            noteMatchesCriterion(file, state.criteria[i])
        );
    }

    const numbers = [];

    for (const file of files) {

        for (
            const value
            of getPropertyValues(
                file,
                criterion.property
            )
        ) {

            const number =
                Number(value);

            if (
                !Number.isNaN(number)
            ) {

                numbers.push(number);
            }
        }
    }

    if (!numbers.length) {
        return;
    }

    const dataMin =
        Math.min(...numbers);

    const dataMax =
        Math.max(...numbers);

    /*
       Valeurs actuelles
    */

    let min =
        criterion.min === null ||
        criterion.min === ""
            ? dataMin
            : Number(criterion.min);

    let max =
        criterion.max === null ||
        criterion.max === ""
            ? dataMax
            : Number(criterion.max);

    /*
       Sécurité
    */

    min =
        Math.max(
            dataMin,
            Math.min(
                min,
                dataMax
            )
        );

    max =
        Math.max(
            dataMin,
            Math.min(
                max,
                dataMax
            )
        );

    if (min > max) {
        min = max;
    }


    const numeric =
        document.createElement(
            "div"
        );

    numeric.className =
        "explorer-numeric";


    const title =
        document.createElement(
            "div"
        );

    title.className =
        "explorer-numeric-title";

    title.textContent =
        "Intervalle de valeurs";

    numeric.appendChild(
        title
    );


    const rangeContainer =
        document.createElement(
            "div"
        );

    rangeContainer.className =
        "explorer-range-container";


    /* --------------------------------------------------------
       MINIMUM
       -------------------------------------------------------- */

    const minGroup =
        document.createElement(
            "div"
        );

    minGroup.className =
        "explorer-range-group";

    const minLabel =
        document.createElement(
            "label"
        );

    minLabel.textContent =
        "Minimum";

    const minSlider =
        document.createElement(
            "input"
        );

    minSlider.type =
        "range";

    minSlider.min =
        dataMin;

    minSlider.max =
        dataMax;

    minSlider.step =
        "any";

    minSlider.value =
        min;

    const minValue =
        document.createElement(
            "span"
        );

    minValue.className =
        "explorer-range-value";

    minValue.textContent =
        min;

    minSlider.oninput = () => {

        let value =
            Number(
                minSlider.value
            );

        if (value > max) {
            value = max;
            minSlider.value =
                value;
        }

        criterion.min =
            value;

        minValue.textContent =
            value;

        renderResultsOnly();
    };

    minGroup.appendChild(
        minLabel
    );

    minGroup.appendChild(
        minSlider
    );

    minGroup.appendChild(
        minValue
    );


    /* --------------------------------------------------------
       MAXIMUM
       -------------------------------------------------------- */

    const maxGroup =
        document.createElement(
            "div"
        );

    maxGroup.className =
        "explorer-range-group";

    const maxLabel =
        document.createElement(
            "label"
        );

    maxLabel.textContent =
        "Maximum";

    const maxSlider =
        document.createElement(
            "input"
        );

    maxSlider.type =
        "range";

    maxSlider.min =
        dataMin;

    maxSlider.max =
        dataMax;

    maxSlider.step =
        "any";

    maxSlider.value =
        max;

    const maxValue =
        document.createElement(
            "span"
        );

    maxValue.className =
        "explorer-range-value";

    maxValue.textContent =
        max;

    maxSlider.oninput = () => {

        let value =
            Number(
                maxSlider.value
            );

        if (value < min) {
            value = min;
            maxSlider.value =
                value;
        }

        criterion.max =
            value;

        maxValue.textContent =
            value;

        renderResultsOnly();
    };

    maxGroup.appendChild(
        maxLabel
    );

    maxGroup.appendChild(
        maxSlider
    );

    maxGroup.appendChild(
        maxValue
    );


    rangeContainer.appendChild(
        minGroup
    );

    rangeContainer.appendChild(
        maxGroup
    );

    numeric.appendChild(
        rangeContainer
    );


    /* VALEURS NUMÉRIQUES */

    const manual =
        document.createElement(
            "div"
        );

    manual.className =
        "explorer-numeric-manual";


    const minInput =
        document.createElement(
            "input"
        );

    minInput.type =
        "number";

    minInput.step =
        "any";

    minInput.value =
        min;

    minInput.title =
        "Minimum";


    const maxInput =
        document.createElement(
            "input"
        );

    maxInput.type =
        "number";

    maxInput.step =
        "any";

    maxInput.value =
        max;

    maxInput.title =
        "Maximum";


    minInput.oninput = () => {

        let value =
            Number(
                minInput.value
            );

        if (
            Number.isNaN(value)
        ) {
            return;
        }

        value =
            Math.max(
                dataMin,
                Math.min(
                    value,
                    max
                )
            );

        criterion.min =
            value;

        minSlider.value =
            value;

        minValue.textContent =
            value;

        renderResultsOnly();
    };


    maxInput.oninput = () => {

        let value =
            Number(
                maxInput.value
            );

        if (
            Number.isNaN(value)
        ) {
            return;
        }

        value =
            Math.max(
                min,
                Math.min(
                    value,
                    dataMax
                )
            );

        criterion.max =
            value;

        maxSlider.value =
            value;

        maxValue.textContent =
            value;

        renderResultsOnly();
    };


    manual.appendChild(
        minInput
    );

    manual.appendChild(
        document.createTextNode(
            " → "
        )
    );

    manual.appendChild(
        maxInput
    );

    numeric.appendChild(
        manual
    );

    container.appendChild(
        numeric
    );
}


/* ============================================================
   TRI
   ============================================================ */

function sortFiles(files) {

    const result = [...files];

    result.sort((a, b) => {

        let va;
        let vb;

        switch (state.sortField) {
            case "path":
                va = a.path; vb = b.path; break;
            case "created":
                va = a.stat.ctime; vb = b.stat.ctime; break;
            case "modified":
                va = a.stat.mtime; vb = b.stat.mtime; break;
            case "name":
                va = a.basename; vb = b.basename; break;
            default:
                va = getFrontmatter(a)[state.sortField];
                vb = getFrontmatter(b)[state.sortField];
                break;
        }

        const aEmpty = va === null || va === undefined || displayValue(va).trim() === "";
        const bEmpty = vb === null || vb === undefined || displayValue(vb).trim() === "";
        if (aEmpty && bEmpty) return 0;
        if (aEmpty) return 1;
        if (bEmpty) return -1;

        const aText = displayValue(va);
        const bText = displayValue(vb);
        const aNumber = Number(aText);
        const bNumber = Number(bText);
        const bothNumeric = !Number.isNaN(aNumber) && !Number.isNaN(bNumber);

        const comparison = bothNumeric
            ? aNumber - bNumber
            : aText.localeCompare(bText, "fr", { numeric: true, sensitivity: "base" });

        return state.sortDirection === "asc" ? comparison : -comparison;
    });

    return result;
}

/* ============================================================
   RENDU DES RÉSULTATS UNIQUEMENT
   ============================================================ */

async function renderResultsOnly() {

    const results =
        root.querySelector(
            ".explorer-results"
        );

    if (!results) {
        return;
    }

    const files =
        getFilteredFiles();

    await renderResults(
        results,
        files
    );
}


/* ============================================================
   RÉSULTATS
   ============================================================ */

async function renderResults(
    container,
    files
) {

    container.innerHTML = "";

    const count =
        document.createElement(
            "div"
        );

    count.className =
        "explorer-result-count";

    count.textContent =
        `${files.length} note(s)`;

    container.appendChild(
        count
    );

    const sorted =
        sortFiles(files);

    for (const file of sorted) {

        const result =
            document.createElement(
                "div"
            );

        result.className =
            "explorer-result";


        /* IMAGE */

        if (
            state.selectedImageProperty
        ) {

            const imageContainer =
                document.createElement(
                    "div"
                );

            imageContainer.className =
                "explorer-result-image-container";

            renderNoteImage(
                file,
                imageContainer
            );

            result.appendChild(
                imageContainer
            );
        }


        /* CONTENU */

        const content =
            document.createElement(
                "div"
            );

        content.className =
            "explorer-result-content";


        const title =
            document.createElement(
                "div"
            );

        title.className =
            "explorer-result-title";

        title.textContent =
            file.basename;

        content.appendChild(
            title
        );


        const path =
            document.createElement(
                "div"
            );

        path.className =
            "explorer-result-path";

        path.textContent =
            file.path;

        content.appendChild(
            path
        );


        const dates =
            document.createElement(
                "div"
            );

        dates.className =
            "explorer-result-dates";

        dates.textContent =
            `Créée : ${new Date(
                file.stat.ctime
            ).toLocaleString("fr-FR")} · ` +
            `Modifiée : ${new Date(
                file.stat.mtime
            ).toLocaleString("fr-FR")}`;

        content.appendChild(
            dates
        );


        /* ACTIONS */

        const actions =
            document.createElement(
                "div"
            );

        actions.className =
            "explorer-result-actions";


        const openButton =
            document.createElement(
                "button"
            );

        openButton.textContent =
            "Ouvrir la note";

        openButton.onclick =
            async () => {

                await app.workspace
                    .getLeaf(false)
                    .openFile(file);
            };

        actions.appendChild(
            openButton
        );


        const contentButton =
            document.createElement(
                "button"
            );

        contentButton.textContent =
            "Afficher le contenu";

        actions.appendChild(
            contentButton
        );

        content.appendChild(
            actions
        );


        /* DÉTAILS */

        const details =
            document.createElement(
                "div"
            );

        details.className =
            "explorer-result-details";

        details.style.display =
            "none";


        contentButton.onclick =
            async () => {

                const visible =
                    details.style.display !==
                    "none";

                details.style.display =
                    visible
                        ? "none"
                        : "block";

                if (
                    visible ||
                    details.dataset.loaded
                ) {
                    return;
                }

                details.dataset.loaded =
                    "true";


                /* PROPRIÉTÉS */

                const fmTitle =
                    document.createElement(
                        "h4"
                    );

                fmTitle.textContent =
                    "Propriétés";

                details.appendChild(
                    fmTitle
                );

                const fm =
                    getFrontmatter(file);

                for (
                    const [key, value]
                    of Object.entries(fm)
                ) {

                    const property =
                        document.createElement(
                            "div"
                        );

                    property.className =
                        "explorer-property";

                    const name =
                        document.createElement(
                            "strong"
                        );

                    name.className =
                        "explorer-property-name";

                    name.textContent =
                        key;

                    const val =
                        document.createElement(
                            "span"
                        );

                    val.className =
                        "explorer-property-value";

                    val.textContent =
                        displayValue(value);

                    property.appendChild(
                        name
                    );

                    property.appendChild(
                        val
                    );

                    details.appendChild(
                        property
                    );
                }


                /* MARKDOWN */

                const mdTitle =
                    document.createElement(
                        "h4"
                    );

                mdTitle.textContent =
                    "Markdown";

                details.appendChild(
                    mdTitle
                );

                const markdown =
                    document.createElement(
                        "div"
                    );

                markdown.className =
                    "explorer-markdown";

                details.appendChild(
                    markdown
                );

                const body =
                    await app.vault.read(
                        file
                    );

                const markdownBody =
                    body.replace(
                        /^---\s*\n[\s\S]*?\n---\s*\n?/,
                        ""
                    );


                if (
                    typeof MarkdownRenderer !==
                        "undefined" &&
                    typeof Component !==
                        "undefined"
                ) {

                    try {

                        await MarkdownRenderer.render(
                            app,
                            markdownBody,
                            markdown,
                            file.path,
                            new Component()
                        );

                    } catch (error) {

                        const pre =
                            document.createElement(
                                "pre"
                            );

                        pre.textContent =
                            markdownBody;

                        markdown.appendChild(
                            pre
                        );
                    }

                } else {

                    const pre =
                        document.createElement(
                            "pre"
                        );

                    pre.textContent =
                        markdownBody;

                    markdown.appendChild(
                        pre
                    );
                }
            };


        content.appendChild(
            details
        );

        result.appendChild(
            content
        );

        container.appendChild(
            result
        );
    }
}


/* ============================================================
   CSS
   ============================================================ */

const style =
    document.createElement(
        "style"
    );

style.textContent = `

.explorer {
    width: 100%;
    box-sizing: border-box;
}

.explorer *,
.explorer *::before,
.explorer *::after {
    box-sizing: border-box;
}


/* ============================================================
   BARRE D'OUTILS
   ============================================================ */

.explorer-toolbar {
    display: flex !important;
    flex-direction: row !important;
    flex-wrap: wrap !important;
    justify-content: flex-start !important;
    align-items: flex-end !important;
    width: 100% !important;
    margin: 0 0 16px 0 !important;
    padding: 12px !important;
    gap: 10px !important;
    text-align: left !important;
    float: none !important;

    border: 1px solid
        var(--background-modifier-border);

    border-radius: 8px;
}

.explorer-toolbar-group {
    display: flex !important;
    flex: 0 0 auto !important;
    flex-direction: column !important;
    align-items: flex-start !important;
    gap: 4px;
}

.explorer-toolbar-group label {
    font-size: 0.85em;
    font-weight: 600;
}

.explorer-toolbar select,
.explorer-toolbar input {
    min-width: 170px;
}

.explorer-toolbar input[type="search"] {
    min-width: 240px;
}

.explorer-buttons {
    display: flex !important;
    flex-wrap: wrap !important;
    gap: 6px;
}


/* ============================================================
   SECTIONS
   ============================================================ */

.explorer-section {
    margin: 18px 0;
}

.explorer-section-title {
    font-size: 1.1em;
    font-weight: 700;
    margin-bottom: 8px;
}


/* ============================================================
   CRITÈRES
   ============================================================ */

.explorer-criteria {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.explorer-criterion {
    border: 1px solid
        var(--background-modifier-border);

    border-radius: 8px;
    padding: 10px;
}

.explorer-criterion-header {
    display: flex;
    justify-content: space-between;
    align-items: center;
    margin-bottom: 10px;
}

.explorer-criterion-body {
    display: flex;
    flex-direction: column;
    gap: 10px;
}

.explorer-criterion-body > select {
    width: 100%;
    max-width: 400px;
}


/* ============================================================
   NULL / NON NULL
   ============================================================ */

.explorer-null-options {
    display: flex;
    flex-wrap: wrap;
    gap: 14px;

    padding: 8px 10px;

    border: 1px solid
        var(--background-modifier-border);

    border-radius: 6px;

    background:
        var(--background-secondary);
}

.explorer-check-label {
    display: flex;
    align-items: center;
    gap: 6px;
    cursor: pointer;
}


/* ============================================================
   AVEC / SANS
   ============================================================ */

.explorer-text-forms {
    display: grid;
    grid-template-columns: 1fr 1fr;
    gap: 14px;
}

@media (max-width: 800px) {
    .explorer-text-forms {
        grid-template-columns: 1fr;
    }
}

.explorer-value-form {
    border: 1px solid
        var(--background-modifier-border);

    border-radius: 7px;
    padding: 10px;
}

.explorer-value-form-title {
    font-weight: 700;
    margin-bottom: 8px;
}

.explorer-autocomplete {
    position: relative;
}

.explorer-autocomplete-input {
    width: 100%;
}

.explorer-autocomplete-list {
    max-height: 220px;
    overflow-y: auto;

    margin-top: 4px;

    border: 1px solid
        var(--background-modifier-border);

    border-radius: 6px;
}

.explorer-autocomplete-item {
    display: flex;
    justify-content: space-between;
    gap: 10px;

    padding: 6px 8px;

    cursor: pointer;
}

.explorer-autocomplete-item:hover {
    background:
        var(--background-modifier-hover);
}

.explorer-autocomplete-count {
    opacity: 0.65;
    font-size: 0.85em;
}

.explorer-selected-values {
    display: flex;
    flex-wrap: wrap;
    gap: 5px;
    margin-top: 8px;
}

.explorer-selected-value {
    display: inline-flex;
    align-items: center;
    gap: 4px;

    padding: 3px 7px;

    border-radius: 12px;

    background:
        var(--interactive-accent);

    color:
        var(--text-on-accent);

    font-size: 0.85em;
}

.explorer-selected-value-remove {
    border: none;
    background: transparent;
    color: inherit;
    cursor: pointer;
    padding: 0;
}

.explorer-with-all {
    display: flex;
    align-items: center;
    gap: 6px;
    margin-top: 10px;
    font-size: 0.9em;
}


/* ============================================================
   NUMÉRIQUE / SLIDER
   ============================================================ */

.explorer-numeric {
    padding: 10px;

    border: 1px solid
        var(--background-modifier-border);

    border-radius: 7px;
}

.explorer-numeric-title {
    font-weight: 700;
    margin-bottom: 12px;
}

.explorer-range-container {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.explorer-range-group {
    display: grid;

    grid-template-columns:
        80px minmax(150px, 1fr) 80px;

    align-items: center;

    gap: 10px;
}

.explorer-range-group label {
    font-weight: 600;
}

.explorer-range-group input[type="range"] {
    width: 100%;
}

.explorer-range-value {
    text-align: right;
    font-family: var(--font-monospace);
}

.explorer-numeric-manual {
    display: flex;
    align-items: center;
    gap: 5px;
    margin-top: 12px;
}

.explorer-numeric-manual input {
    width: 120px;
}


/* ============================================================
   STATISTIQUES
   ============================================================ */

.explorer-statistics {
    border: 1px solid
        var(--background-modifier-border);

    border-radius: 8px;

    overflow: hidden;
}

.explorer-statistics-header {
    display: flex;
    align-items: center;
    gap: 8px;

    padding: 10px 12px;

    cursor: pointer;

    user-select: none;

    font-weight: 700;

    background:
        var(--background-secondary);
}

.explorer-statistics-header:hover {
    background:
        var(--background-modifier-hover);
}

.explorer-statistics-arrow {
    display: inline-block;
    width: 16px;
}

.explorer-statistics-content {
    padding: 12px;
}

.explorer-statistics-summary {
    margin-bottom: 12px;
    opacity: 0.75;
}

.explorer-stat-criterion {
    border-bottom: 1px solid
        var(--background-modifier-border);

    padding: 8px 0;
}

.explorer-stat-criterion:last-child {
    border-bottom: none;
}

.explorer-stat-criterion-title {
    font-weight: 700;
    margin-bottom: 6px;
}

.explorer-stat-lines {
    display: flex;
    flex-direction: column;
    gap: 5px;
}

.explorer-stat-button,
.explorer-stat-value {
    width: fit-content;
    text-align: left;
    cursor: pointer;
}

.explorer-stat-details {
    margin: 5px 0 8px 18px;

    padding: 7px;

    border-left: 2px solid
        var(--background-modifier-border);
}

.explorer-stat-note {
    padding: 5px 0;
}

.explorer-stat-note-name {
    font-weight: 600;
}

.explorer-stat-note-path {
    opacity: 0.65;
    font-size: 0.8em;
}

.explorer-stat-note-values {
    margin-top: 2px;
    font-size: 0.85em;
}

.explorer-stat-value {
    display: block;
    margin: 2px 0;
}

.explorer-stat-empty {
    opacity: 0.65;
    padding: 8px 0;
}


/* ============================================================
   RÉSULTATS
   ============================================================ */

.explorer-results {
    display: flex;
    flex-direction: column;
    gap: 12px;
}

.explorer-result-count {
    font-weight: 700;
    margin-bottom: 4px;
}

.explorer-result {
    display: flex;
    gap: 14px;

    border: 1px solid
        var(--background-modifier-border);

    border-radius: 8px;

    padding: 12px;
}

.explorer-result-image-container {
    width: 150px;
    min-width: 150px;

    display: flex;
    align-items: flex-start;
    justify-content: center;
}

.explorer-result-image {
    max-width: 150px;
    max-height: 150px;

    object-fit: contain;

    border-radius: 6px;
}

.explorer-result-content {
    flex: 1;
    min-width: 0;
}

.explorer-result-title {
    font-size: 1.05em;
    font-weight: 700;
}

.explorer-result-path {
    font-size: 0.82em;
    opacity: 0.65;
    word-break: break-all;
}

.explorer-result-dates {
    font-size: 0.8em;
    opacity: 0.65;
    margin: 4px 0 8px;
}

.explorer-result-actions {
    display: flex;
    flex-wrap: wrap;
    gap: 6px;
    margin-bottom: 8px;
}

.explorer-result-details {
    padding-top: 8px;

    border-top: 1px solid
        var(--background-modifier-border);
}

.explorer-property {
    display: flex;
    gap: 10px;
    padding: 3px 0;
}

.explorer-property-name {
    min-width: 150px;
}

.explorer-property-value {
    word-break: break-word;
}

.explorer-markdown {
    margin-top: 8px;
}

.explorer-markdown pre {
    white-space: pre-wrap;
}

.explorer-empty {
    opacity: 0.65;
    padding: 8px 0;
}

`;


/* ============================================================
   INSTALLATION DU CSS
   ============================================================ */

root.appendChild(style);


/* ============================================================
   RENDU GLOBAL
   ============================================================ */

async function renderAll() {

    let appContainer =
        root.querySelector(
            ".explorer"
        );

    if (!appContainer) {

        appContainer =
            document.createElement(
                "div"
            );

        appContainer.className =
            "explorer";

        root.appendChild(
            appContainer
        );
    }

    appContainer.innerHTML = "";


    /* ========================================================
       BARRE D'OUTILS
       ======================================================== */

    const toolbar =
        document.createElement(
            "div"
        );

    toolbar.className =
        "explorer-toolbar";


    /* DOSSIER */

    const folderGroup =
        document.createElement(
            "div"
        );

    folderGroup.className =
        "explorer-toolbar-group";

    const folderLabel =
        document.createElement(
            "label"
        );

    folderLabel.textContent =
        "Dossier";

    const folderSelect =
        document.createElement(
            "select"
        );

    for (
        const folder
        of getFolders()
    ) {

        const option =
            document.createElement(
                "option"
            );

        option.value =
            folder;

        option.textContent =
            folder ||
            "(Tout le coffre)";

        option.selected =
            folder ===
            state.selectedFolder;

        folderSelect.appendChild(
            option
        );
    }

    folderSelect.onchange = () => {

        state.selectedFolder =
            folderSelect.value;

        const properties =
            getProperties(
                getFilesForFolder(
                    state.selectedFolder
                )
            );

        if (
            !properties.includes(
                state.selectedImageProperty
            )
        ) {

            state.selectedImageProperty =
                "";
        }

        renderAll();
    };

    folderGroup.appendChild(
        folderLabel
    );

    folderGroup.appendChild(
        folderSelect
    );

    toolbar.appendChild(
        folderGroup
    );


    /* IMAGE */

    const imageGroup =
        document.createElement(
            "div"
        );

    imageGroup.className =
        "explorer-toolbar-group";

    const imageLabel =
        document.createElement(
            "label"
        );

    imageLabel.textContent =
        "Propriété image";

    const imageSelect =
        document.createElement(
            "select"
        );

    const noImage =
        document.createElement(
            "option"
        );

    noImage.value =
        "";

    noImage.textContent =
        "Aucune image";

    imageSelect.appendChild(
        noImage
    );

    const folderFiles =
        getFilesForFolder(
            state.selectedFolder
        );

    const properties =
        getProperties(
            folderFiles
        );

    for (
        const property
        of properties
    ) {

        const option =
            document.createElement(
                "option"
            );

        option.value =
            property;

        option.textContent =
            property;

        option.selected =
            property ===
            state.selectedImageProperty;

        imageSelect.appendChild(
            option
        );
    }

    imageSelect.onchange = () => {

        state.selectedImageProperty =
            imageSelect.value;

        renderAll();
    };

    imageGroup.appendChild(
        imageLabel
    );

    imageGroup.appendChild(
        imageSelect
    );

    toolbar.appendChild(
        imageGroup
    );


    /* RECHERCHE */

    const searchGroup =
        document.createElement(
            "div"
        );

    searchGroup.className =
        "explorer-toolbar-group";

    const searchLabel =
        document.createElement(
            "label"
        );

    searchLabel.textContent =
        "Recherche";

    const searchInput =
        document.createElement(
            "input"
        );

    searchInput.type =
        "search";

    searchInput.placeholder =
        "Nom, chemin ou propriété...";

    searchInput.value =
        state.search;

    searchInput.oninput = () => {

        state.search =
            searchInput.value;

        renderAll();
    };

    searchGroup.appendChild(
        searchLabel
    );

    searchGroup.appendChild(
        searchInput
    );

    toolbar.appendChild(
        searchGroup
    );


    /* TRI */

    const sortGroup =
        document.createElement(
            "div"
        );

    sortGroup.className =
        "explorer-toolbar-group";

    const sortLabel =
        document.createElement(
            "label"
        );

    sortLabel.textContent =
        "Tri";

    const sortSelect =
        document.createElement(
            "select"
        );

    const sortOptions = [
        ["name", "Nom"],
        ["path", "Chemin"],
        ["created", "Date de création"],
        ["modified", "Date de modification"]
    ];

    const sortProperties = getProperties(
        getFilesForFolder(state.selectedFolder)
    );

    for (const property of sortProperties) {
        sortOptions.push([property, property]);
    }

    for (
        const [value, label]
        of sortOptions
    ) {

        const option =
            document.createElement(
                "option"
            );

        option.value =
            value;

        option.textContent =
            label;

        option.selected =
            state.sortField ===
            value;

        sortSelect.appendChild(
            option
        );
    }

    sortSelect.onchange = () => {

        state.sortField =
            sortSelect.value;

        renderAll();
    };

    sortGroup.appendChild(
        sortLabel
    );

    sortGroup.appendChild(
        sortSelect
    );

    toolbar.appendChild(
        sortGroup
    );


    /* ORDRE */

    const directionGroup =
        document.createElement(
            "div"
        );

    directionGroup.className =
        "explorer-toolbar-group";

    const directionLabel =
        document.createElement(
            "label"
        );

    directionLabel.textContent =
        "Ordre";

    const directionSelect =
        document.createElement(
            "select"
        );

    const asc =
        document.createElement(
            "option"
        );

    asc.value =
        "asc";

    asc.textContent =
        "Croissant";

    asc.selected =
        state.sortDirection ===
        "asc";

    const desc =
        document.createElement(
            "option"
        );

    desc.value =
        "desc";

    desc.textContent =
        "Décroissant";

    desc.selected =
        state.sortDirection ===
        "desc";

    directionSelect.appendChild(
        asc
    );

    directionSelect.appendChild(
        desc
    );

    directionSelect.onchange = () => {

        state.sortDirection =
            directionSelect.value;

        renderAll();
    };

    directionGroup.appendChild(
        directionLabel
    );

    directionGroup.appendChild(
        directionSelect
    );

    toolbar.appendChild(
        directionGroup
    );


    /* TRI DES VALEURS */

    const valueSortGroup =
        document.createElement(
            "div"
        );

    valueSortGroup.className =
        "explorer-toolbar-group";

    const valueSortLabel =
        document.createElement(
            "label"
        );

    valueSortLabel.textContent =
        "Tri des valeurs";

    const valueSortSelect =
        document.createElement(
            "select"
        );

    const countOption =
        document.createElement(
            "option"
        );

    countOption.value =
        "count";

    countOption.textContent =
        "Occurrences";

    countOption.selected =
        state.valueSort ===
        "count";

    const alphaOption =
        document.createElement(
            "option"
        );

    alphaOption.value =
        "alpha";

    alphaOption.textContent =
        "Alphabétique";

    alphaOption.selected =
        state.valueSort ===
        "alpha";

    valueSortSelect.appendChild(
        countOption
    );

    valueSortSelect.appendChild(
        alphaOption
    );

    valueSortSelect.onchange = () => {

        state.valueSort =
            valueSortSelect.value;

        renderAll();
    };

    valueSortGroup.appendChild(
        valueSortLabel
    );

    valueSortGroup.appendChild(
        valueSortSelect
    );

    toolbar.appendChild(
        valueSortGroup
    );


    appContainer.appendChild(
        toolbar
    );


    /* ========================================================
       CRITÈRES
       ======================================================== */

    const criteriaSection =
        document.createElement(
            "div"
        );

    criteriaSection.className =
        "explorer-section";

    const criteriaTitle =
        document.createElement(
            "div"
        );

    criteriaTitle.className =
        "explorer-section-title";

    criteriaTitle.textContent =
        "Critères";

    criteriaSection.appendChild(
        criteriaTitle
    );


    const criteriaButtons =
        document.createElement(
            "div"
        );

    criteriaButtons.className =
        "explorer-buttons";

    const addCriterion =
        document.createElement(
            "button"
        );

    addCriterion.textContent =
        "+ Ajouter un critère";

    addCriterion.onclick = () => {

        state.criteria.push(
            createCriterion()
        );

        renderAll();
    };

    criteriaButtons.appendChild(
        addCriterion
    );

    criteriaSection.appendChild(
        criteriaButtons
    );


    const criteria =
        document.createElement(
            "div"
        );

    criteria.className =
        "explorer-criteria";

    renderCriteria(
        criteria
    );

    criteriaSection.appendChild(
        criteria
    );

    appContainer.appendChild(
        criteriaSection
    );


    /* ========================================================
       STATISTIQUES REPLIABLES
       ======================================================== */

    const statisticsSection =
        document.createElement(
            "div"
        );

    statisticsSection.className =
        "explorer-section explorer-statistics";


    const statisticsHeader =
        document.createElement(
            "div"
        );

    statisticsHeader.className =
        "explorer-statistics-header";


    const arrow =
        document.createElement(
            "span"
        );

    arrow.className =
        "explorer-statistics-arrow";

    arrow.textContent =
        state.statisticsOpen
            ? "▼"
            : "▶";


    const headerText =
        document.createElement(
            "span"
        );

    headerText.textContent =
        "Statistiques";


    const filteredFiles =
        getFilteredFiles();


    const summary =
        document.createElement(
            "span"
        );

    summary.style.opacity =
        "0.65";

    summary.textContent =
        `(${filteredFiles.length} note(s))`;


    statisticsHeader.appendChild(
        arrow
    );

    statisticsHeader.appendChild(
        headerText
    );

    statisticsHeader.appendChild(
        summary
    );


    statisticsHeader.onclick = () => {

        state.statisticsOpen =
            !state.statisticsOpen;

        renderAll();
    };


    statisticsSection.appendChild(
        statisticsHeader
    );


    if (
        state.statisticsOpen
    ) {

        const statisticsContent =
            document.createElement(
                "div"
            );

        statisticsContent.className =
            "explorer-statistics-content";

        renderStatistics(
            statisticsContent,
            filteredFiles
        );

        statisticsSection.appendChild(
            statisticsContent
        );
    }


    appContainer.appendChild(
        statisticsSection
    );


    /* ========================================================
       RÉSULTATS
       ======================================================== */

    const resultsSection =
        document.createElement(
            "div"
        );

    resultsSection.className =
        "explorer-section";


    const resultsTitle =
        document.createElement(
            "div"
        );

    resultsTitle.className =
        "explorer-section-title";

    resultsTitle.textContent =
        "Résultats";

    resultsSection.appendChild(
        resultsTitle
    );


    const results =
        document.createElement(
            "div"
        );

    results.className =
        "explorer-results";

    resultsSection.appendChild(
        results
    );

    appContainer.appendChild(
        resultsSection
    );


    await renderResults(
        results,
        filteredFiles
    );
}


/* ============================================================
   LANCEMENT
   ============================================================ */

await renderAll();
```