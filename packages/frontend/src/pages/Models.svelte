<script lang="ts">
import type { ModelInfo } from '@shared/src/models/IModelInfo';
import NavPage from '../lib/NavPage.svelte';
import Table from '../lib/table/Table.svelte';
import { Column, Row } from '../lib/table/table';
import { modelsInfo } from '../stores/modelsInfo';
import ModelColumnName from '../lib/table/model/ModelColumnName.svelte';
import ModelColumnLabels from '../lib/table/model/ModelColumnLabels.svelte';
import type { Task } from '@shared/src/models/ITask';
import TasksProgress from '/@/lib/progress/TasksProgress.svelte';
import Card from '/@/lib/Card.svelte';
import { onMount } from 'svelte';
import ModelColumnSize from '../lib/table/model/ModelColumnSize.svelte';
import ModelColumnAge from '../lib/table/model/ModelColumnAge.svelte';
import ModelColumnActions from '../lib/table/model/ModelColumnActions.svelte';
import Tab from '/@/lib/Tab.svelte';
import Route from '/@/Route.svelte';
import { tasks } from '/@/stores/tasks';
import { catalog } from '../stores/catalog';
import ModelColumnIcon from '../lib/table/model/ModelColumnIcon.svelte';
import Button from '../lib/button/Button.svelte';
import { router } from 'tinro';

const columns: Column<ModelInfo>[] = [
  new Column<ModelInfo>('', { width: '40px', renderer: ModelColumnIcon }),
  new Column<ModelInfo>('Name', {
    width: '3fr',
    renderer: ModelColumnName,
    comparator: (a, b) => b.name.localeCompare(a.name),
  }),
  new Column<ModelInfo>('Size', {
    width: '50px',
    renderer: ModelColumnSize,
    comparator: (a, b) => (a.file?.size ?? 0) - (b.file?.size ?? 0),
  }),
  new Column<ModelInfo>('Age', {
    width: '70px',
    renderer: ModelColumnAge,
    comparator: (a, b) => (a.file?.creation?.getTime() ?? 0) - (b.file?.creation?.getTime() ?? 0),
  }),
  new Column<ModelInfo>('', { width: '225px', align: 'right', renderer: ModelColumnLabels }),
  new Column<ModelInfo>('Actions', { align: 'right', width: '120px', renderer: ModelColumnActions }),
];
const row = new Row<ModelInfo>({});

let loading: boolean = true;

let pullingTasks: Task[] = [];
let models: ModelInfo[] = [];

// filtered mean, we remove the models that are being downloaded
let filteredModels: ModelInfo[] = [];

$: localModels = filteredModels.filter(model => model.file);
$: remoteModels = filteredModels.filter(model => !model.file);

function filterModels(): void {
  // Let's collect the models we do not want to show (loading, error).
  const modelsId: string[] = pullingTasks.reduce((previousValue, currentValue) => {
    if (currentValue.labels !== undefined) {
      previousValue.push(currentValue.labels['model-pulling']);
    }
    return previousValue;
  }, [] as string[]);
  filteredModels = models.filter(model => !modelsId.includes(model.id));
}

onMount(() => {
  // Subscribe to the tasks store
  const tasksUnsubscribe = tasks.subscribe(value => {
    // Filter out duplicates
    const modelIds = new Set<string>();
    pullingTasks = value.reduce((filtered: Task[], task: Task) => {
      if (
        task.state === 'loading' &&
        task.labels !== undefined &&
        'model-pulling' in task.labels &&
        !modelIds.has(task.labels['model-pulling'])
      ) {
        modelIds.add(task.labels['model-pulling']);
        filtered.push(task);
      }
      return filtered;
    }, []);

    loading = false;
    filterModels();
  });

  // Subscribe to the models store
  const localModelsUnsubscribe = modelsInfo.subscribe(value => {
    models = value;
    filterModels();
  });

  return () => {
    tasksUnsubscribe();
    localModelsUnsubscribe();
  };
});

async function importModel() {
  router.goto('/models/import');
}
</script>

<NavPage title="Models" searchEnabled="{false}" loading="{loading}">
  <svelte:fragment slot="tabs">
    <Tab title="All" url="models" />
    <Tab title="Downloaded" url="models/downloaded" />
    <Tab title="Available" url="models/available" />
  </svelte:fragment>
  <svelte:fragment slot="additional-actions">
    <Button on:click="{importModel}" aria-label="Import Models">Import</Button>
  </svelte:fragment>
  <svelte:fragment slot="content">
    <div slot="content" class="flex flex-col min-w-full min-h-full">
      <div class="min-w-full min-h-full flex-1">
        <div class="mt-4 px-5 space-y-5">
          {#if !loading}
            {#if pullingTasks.length > 0}
              <Card classes="bg-charcoal-800 mt-4">
                <div slot="content" class="text-base font-normal p-2 w-full">
                  <div class="text-base mb-2">Downloading models</div>
                  <TasksProgress tasks="{pullingTasks}" />
                </div>
              </Card>
            {/if}

            <!-- All models -->
            <Route path="/">
              {#if filteredModels.length > 0}
                <Table kind="model" data="{filteredModels}" columns="{columns}" row="{row}"></Table>
              {:else}
                <div role="status">There is no model yet</div>
              {/if}
            </Route>

            <!-- Downloaded models -->
            <Route path="/downloaded">
              {#if localModels.length > 0}
                <Table kind="model" data="{localModels}" columns="{columns}" row="{row}"></Table>
              {:else}
                <div role="status">There is no model yet</div>
              {/if}
            </Route>

            <!-- Available models (from catalogs)-->
            <Route path="/available">
              {#if remoteModels.length > 0}
                <Table kind="model" data="{remoteModels}" columns="{columns}" row="{row}"></Table>
              {:else}
                <div role="status">There is no model yet</div>
              {/if}
            </Route>
          {/if}
        </div>
      </div>
    </div>
  </svelte:fragment>
</NavPage>
