<template>
  <div id="controls-container">
    <RouterLink :to="{ name: 'home' }" style="margin-right: 40px;">Home</RouterLink>
    <span v-if="widgets.savedData.size === 0">&lt;No Entries&gt;</span>
    <template v-else>
      <label for="entry-select">Entry</label>
      <select id="entry-select" v-model.number="selectedIdx">
        <option v-for="[i, name] of entries.entries()" :key="i" :value="i">{{ name }}</option>
      </select>
      <button @click="deleteData">Delete</button>
      <button @click="downloadData">Download</button>
      <button @click="uploadToDrive" >Upload To Drive
        <div id="g_id_onload"
          data-client_id="344124639348-l0ld2jh9h6no4ja3mpei30j6b9nn6o8h.apps.googleusercontent.com"
          data-login_uri="https://turbotrojans-3302.github.io"
          data-your_own_param_1_to_login="5"
          data-your_own_param_2_to_login="5">
        </div>
      </button>
      <button @click="clearData">Clear All</button>
    </template>
  </div>
  <div class="table-container">
    <span v-if="selectedEntry === undefined">No Data</span>
    <InspectorTable v-else v-model="selectedRecords" :data="selectedEntry" />
  </div>
  <a :hidden="true" :download="entries[selectedIdx]" ref="downloadLink"></a>
</template>

<script setup lang="ts">
import { Name } from "ajv";
import InspectorTable from "./InspectorTable.vue";
import { useWidgetsStore } from "@/common/stores";
// const require = createRequire(import.meta.url);



const widgets = useWidgetsStore();
let selectedIdx = $ref(0); // The index of the entry selected in the combobox
let folderId = "";
const downloadLink = $ref<HTMLAnchorElement>();
const selectedRecords = $ref(new Set<number>());
const hasSelectedRecords = $computed(() => selectedRecords.size > 0);

const entries = $computed(() => [...widgets.savedData.keys()]); // The entries in local storage
const selectedEntry = $computed(() => widgets.savedData.get(entries[selectedIdx])); // The selected entry

// Filters records in the selected entry based on the user selection.
// If there are no records selected, the filter directly uses the given state, returning either all or no records.
const filterRecords = (state: boolean) => (selectedEntry === undefined)
  ? []
  : selectedEntry.values.filter((_v, i) => hasSelectedRecords ? (selectedRecords.has(i) === state) : state);

function deleteData() {
  if (selectedEntry === undefined) return;

  if (!confirm(`Delete ${hasSelectedRecords ? "the selected" : "all"} records in this entry permanently?`)) return;

  // Discard out the selected records
  // If there are none selected, they are all deleted
  selectedEntry.values = filterRecords(false);

  selectedRecords.clear();
}

function downloadData() {
  if (selectedEntry === undefined) return;
  if (downloadLink === undefined) return; // Make sure the link exists

  // Generate the download link for the selected records, then trigger the download
  // If there are no records selected, they will all be included in the generated file
  downloadLink.href = widgets.makeDownloadLink({ header: selectedEntry.header, values: filterRecords(true) });
  downloadLink.click();
}

// uploads CSV files to Google Drive(does nothing for now, sorry!)
// link to example: https://developers.google.com/drive/api/guides/folder#node.js
// link to stackoverflow question: https://stackoverflow.com/questions/51584732/create-folder-and-upload-file-to-google-drive-from-typescript-cannot-compile
// link to node.js blog: https://nodejs.org/en/blog/announcements/v22-release-announce
async function uploadToDrive(){

  const fs = await import("fs");
  const{GoogleAuth} = await import("google-auth-library");
  const{google} = await import("googleapis")

  // get credentials and build service
  // get proper auth mechanism?(not sure what that means, look up later)
  const auth = new GoogleAuth({scopes: "https://www.googleapis.com/auth/drive",});
  const service = google.drive({version: "v3", auth});

  //set folderId and upload csv
  folderId = "1HyC6zKH98n0OzDhmhKWJkupdxY-Knd1k";

  // copied code from downladData finction
  if (selectedEntry === undefined) return;
  if (downloadLink === undefined) return; // Make sure the link exists
  downloadLink.href = widgets.makeDownloadLink({ header: selectedEntry.header, values: filterRecords(true) });

  const fileMetadata = {
    Name: downloadLink.href,
    parents: [folderId]
  };

  const media = {
    mimeType: "text/csv",
    body: fs.createReadStream(`files/${Name}`),
  };

  try {
    const file = await service.files.create({
      requestBody: fileMetadata,
      media: media,
      fields: "id",
    });
    console.log("File Id:", file.data.id);
    return file.data.id;
  } catch (err) {
    // TODO(developer) - Handle error
    throw err;
  }

}

function clearData() {
  if (!confirm("Clear all saved entries in local storage permanently?")) return;

  widgets.savedData.clear();
  selectedIdx = 0; // Reset selected index
}
</script>

<style>
.table-container {
  overflow: auto;
}

#controls-container>* {
  margin: 4px;
}
</style>
