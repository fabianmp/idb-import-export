<script setup lang="ts">
import { SelectItem } from "@nuxt/ui";
import { openDB } from "idb";

const toast = useToast();

interface Database {
  name: string;
  keyPath: string;
  storeName: string;
  entries: number;
  transformKey: (value: string) => any;
}

function parseDate(value: string) {
  return new Date(Date.parse(value));
}

const databases = ref([
  <Database>{
    name: "Startpage",
    keyPath: "timestamp",
    storeName: "data",
    entries: 0,
    transformKey: parseDate,
  },
  <Database>{
    name: "project-time",
    keyPath: "timestamp",
    storeName: "data",
    entries: 0,
    transformKey: parseDate,
  },
]);

const importData = ref([]);
const clearData = ref(true);
const importFile = ref<File>();

async function createDbConnection(
  database: string,
  keyPath: string,
  store: string = "data",
) {
  const db = await openDB(database, 1, {
    upgrade(db, _oldVersion, _newVersion, _transaction, _event) {
      db.createObjectStore(store, { keyPath: keyPath });
    },
  });
  return db;
}

function sendDownloadFile(basename: string, data: any) {
  const timestamp = new Date().toISOString();
  const jsonContent = encodeURI(
    "data:application/json;charset=utf-8," + JSON.stringify(data),
  ).replaceAll("#", "%23");
  var link = document.createElement("a");
  link.setAttribute("href", jsonContent);
  link.setAttribute("download", `${basename}-${timestamp}.json`);
  document.body.appendChild(link);
  link.click();
  document.body.removeChild(link);
}

async function exportDatabase(database: Database) {
  const db = await createDbConnection(
    database.name,
    database.keyPath,
    database.storeName,
  );
  const data = await db.getAll(database.storeName);
  sendDownloadFile(database.name, data);
  toast.add(<Toast>{
    title: `Exported data for ${database.name} with ${database.entries} entries`,
  });
}

async function importDatabase(database: Database) {
  const db = await createDbConnection(
    database.name,
    database.keyPath,
    database.storeName,
  );
  if (clearData.value) {
    await db.clear(database.storeName);
  }
  for (const entry of importData.value) {
    const parsed = JSON.parse(JSON.stringify(entry));
    parsed[database.keyPath] = database.transformKey(parsed[database.keyPath]);
    await db.add(database.storeName, parsed);
  }
  toast.add(<Toast>{
    title: `Imported ${importData.value.length} entries into ${database.name}`,
  });
  importData.value = [];
  importFile.value = undefined;
}

onMounted(async () => {
  for (const database of databases.value) {
    const db = await createDbConnection(
      database.name,
      database.keyPath,
      database.storeName,
    );
    database.entries = await db.count(database.storeName);
  }

  databaseItems.value = databases.value.map(
    (d) =>
      <SelectItem>{
        label: `${d.name} (${d.entries} entries)`,
        id: d.name,
      },
  );
});

watch(importFile, async (f) => {
  if (f) {
    const text = await f?.text();
    importData.value = JSON.parse(text);
  } else {
    importData.value = [];
  }
});

const databaseItems = ref<SelectItem[]>([]);
const selected = ref(databases.value[0].name);
const database = computed(() =>
  databases.value.find((d) => d.name === selected.value),
);
</script>

<template>
  <Suspense>
    <UApp>
      <UHeader title="idb import export" />
      <UMain>
        <UContainer class="py-5">
          <UCard variant="subtle">
            <template #header>
              Select IndexedDB:
              <USelect
                icon="fa7-solid:database"
                v-model="selected"
                value-key="id"
                :items="databaseItems"
                class="w-80"
              />
            </template>
            <template #default>
              <div class="flex gap-2 w-full" v-if="database">
                <div class="flex flex-col flex-1">
                  <UButton
                    icon="fa7-solid:file-export"
                    :label="`Export ${database.entries} entries`"
                    color="info"
                    @click="exportDatabase(database)"
                  />
                </div>
                <div class="flex flex-col flex-1 gap-2">
                  <UButton
                    icon="fa7-solid:file-import"
                    :label="`Import ${importData.length} entries`"
                    color="success"
                    @click="importDatabase(database)"
                    :disabled="importData.length == 0"
                  />
                  <UFileUpload
                    v-model="importFile"
                    accept="application/json"
                    label="Upload exported IndexedDB file"
                    description="Only JSON format is supported"
                  />
                  <div v-if="importFile">
                    {{ importFile.name }}
                    {{ Math.round(importFile.size / 1024) }} kb
                  </div>
                  <UCheckbox v-model="clearData" label="Clear existing data" />
                </div>
              </div>
            </template>
          </UCard>
        </UContainer>
      </UMain>
    </UApp>
  </Suspense>
</template>
