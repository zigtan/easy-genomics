<script setup lang="ts">
  import { LabUser } from '@easy-genomics/shared-lib/src/app/types/easy-genomics/user-unified';
  import { Laboratory } from '@easy-genomics/shared-lib/src/app/types/easy-genomics/laboratory';
  import {
    LaboratoryRolesEnum,
    LaboratoryRolesEnumSchema,
  } from '@easy-genomics/shared-lib/src/app/types/easy-genomics/roles';
  import { ButtonVariantEnum } from '@FE/types/buttons';
  import { DeletedResponse, EditUserResponse } from '@FE/types/api';
  import { useRunStore, useSeqeraPipelinesStore, useToastStore, useUiStore } from '@FE/stores';
  import useUser from '@FE/composables/useUser';
  import { LaboratoryUserDetails } from '@easy-genomics/shared-lib/src/app/types/easy-genomics/laboratory-user-details';
  import { LaboratoryUser } from '@easy-genomics/shared-lib/src/app/types/easy-genomics/laboratory-user';
  import { v4 as uuidv4 } from 'uuid';
  import { Pipeline as SeqeraPipeline } from '@easy-genomics/shared-lib/src/app/types/nf-tower/nextflow-tower-api';
  import { WorkflowListItem as OmicsWorkflow } from '@aws-sdk/client-omics';
  import { LaboratoryRun } from '@easy-genomics/shared-lib/src/app/types/easy-genomics/laboratory-run';
  import { TableSort } from './EGTable.vue';

  const props = defineProps<{
    superuser?: boolean;
    labId: string;
    initialTab: string | undefined;
  }>();

  const { $api } = useNuxtApp();
  const $router = useRouter();
  const { updateDefaultLab } = useUser();

  const runStore = useRunStore();
  const labStore = useLabsStore();
  const uiStore = useUiStore();
  const userStore = useUserStore();
  const seqeraPipelinesStore = useSeqeraPipelinesStore();
  const omicsWorkflowsStore = useOmicsWorkflowsStore();
  const runsTableRefreshKey = ref(0);

  const { stringSortCompare } = useSort();

  const labUsers = ref<LabUser[]>([]);
  const seqeraPipelines = computed<SeqeraPipeline[]>(() => seqeraPipelinesStore.pipelinesForLab(props.labId));
  const omicsWorkflows = computed<OmicsWorkflow[]>(() => omicsWorkflowsStore.workflowsForLab(props.labId));
  const canAddUsers = computed<boolean>(() => userStore.canAddLabUsers(props.labId));
  const showAddUserModule = ref(false);
  const searchOutput = ref('');
  const runToCancel = ref<LaboratoryRun | null>(null);
  const isCancelDialogOpen = ref<boolean>(false);
  const isMissingPATModalOpen = ref<boolean>(false);
  const isOpen = ref(false);
  const primaryMessage = ref('');
  const userToRemove = ref();
  const missingPAT = ref<boolean>(false);
  const tabIndex = ref(0);
  let intervalId: number | undefined;

  const orgId = computed<string | null>(() => labStore.labs[props.labId].OrganizationId ?? null);
  const lab = computed<Laboratory | null>(() => labStore.labs[props.labId] ?? null);
  const labName = computed<string>(() => lab.value?.Name || '');

  /**
   * Fetch Lab details, pipelines, workflows, runs, and Lab users before component mount and start periodic fetching
   */
  onBeforeMount(async () => {
    await loadLabData();
    await pollFetchLaboratoryRuns();
  });

  onMounted(async () => {
    // set tabIndex according to initialTab prop
    setTabIndex();

    // clean up timeout
    if (intervalId) {
      clearTimeout(intervalId);
    }

    // update most recent lab
    await updateDefaultLab(props.labId);
  });

  onBeforeRouteLeave(() => {
    if (intervalId) {
      clearTimeout(intervalId);
    }
  });

  // Page Tabs

  const tabItems = computed<{ key: string; label: string }[]>(() => {
    const runsTab = { key: 'runs', label: 'Pipeline Runs' };
    const seqeraPipelinesTab = { key: 'seqeraPipelines', label: 'Seqera Pipelines' };
    const omicsWorkflowsTab = { key: 'omicsWorkflows', label: 'HealthOmics Workflows' };
    const usersTab = { key: 'users', label: 'Users' };
    const detailsTab = { key: 'details', label: 'Settings' };

    const seqeraAvailable = lab.value?.NextFlowTowerEnabled && !missingPAT.value;
    const omicsAvailable = lab.value?.AwsHealthOmicsEnabled;

    const items = [];

    if (!props.superuser) {
      if (seqeraAvailable || omicsAvailable) items.push(runsTab);
      if (seqeraAvailable) items.push(seqeraPipelinesTab);
      if (omicsAvailable) items.push(omicsWorkflowsTab);
    }
    items.push(usersTab);
    items.push(detailsTab);

    return items;
  });

  function setTabIndex() {
    const tabMatchIndex = tabItems.value.findIndex((tab) => tab.label === props.initialTab);
    tabIndex.value = tabMatchIndex !== -1 ? tabMatchIndex : 0;
  }

  // Lab Runs Tab

  type LaboratoryRunTableItem = LaboratoryRun & { lastUpdated: string };

  const runsTableColumns = [
    { key: 'RunName', label: 'Run Name', sortable: true },
    { key: 'CreatedAt', label: 'Created At', sortable: true },
    { key: 'lastUpdated', label: 'Last Updated', sortable: true },
    { key: 'Status', label: 'Status', sortable: true },
    { key: 'Owner', label: 'Owner', sortable: true },
    { key: 'actions', label: 'Actions' },
  ];

  const runsTableSort = ref<TableSort>({ column: 'CreatedAt', direction: 'desc' });

  const runsTableFilterMyRunsOnly = ref<boolean>(false);

  const runsTableItems = ref<LaboratoryRunTableItem[]>([]);

  // fetch the runs with BE filtering any time any of the inputs change
  watchEffect(async () => {
    uiStore.setRequestPending('loadLabRuns');

    // without this following line, watchEffect doesn't pick up runsTableSort as a reactive dependency...
    runsTableSort.value;

    // laboratory run polling refresh key
    runsTableRefreshKey.value;

    const filters: any = {};
    if (runsTableFilterMyRunsOnly.value) filters.UserId = userStore.currentUserDetails.id!;

    try {
      runsTableItems.value = (await $api.labs.listLabRuns(props.labId, filters))
        .map((labRun) => ({
          ...labRun,
          lastUpdated: labRun.ModifiedAt ?? labRun.CreatedAt ?? '',
        }))
        .sort((a: any, b: any) =>
          stringSortCompare(
            a[runsTableSort.value.column],
            b[runsTableSort.value.column],
            runsTableSort.value.direction,
          ),
        );
    } finally {
      uiStore.setRequestComplete('loadLabRuns');
    }
  });

  function runsActionItems(run: LaboratoryRun): object[] {
    const buttons: object[][] = [
      [{ label: 'View Details', click: () => viewRunDetails(run) }],
      [{ label: 'View Files', click: () => viewRunDetails(run, 'File Manager') }],
    ];

    if (['SUBMITTED', 'STARTING', 'RUNNING'].includes(run.Status)) {
      buttons.push([{ label: 'Cancel Run', click: () => initCancelRun(run), isHighlighted: true }]);
    }

    return buttons;
  }

  function viewRunDetails(run: LaboratoryRun, tab: string = 'Run Details') {
    $router.push({
      path: `/labs/${props.labId}/run/${run.RunId}`,
      query: { tab },
    });
  }

  function initCancelRun(run: LaboratoryRun) {
    runToCancel.value = run;
    isCancelDialogOpen.value = true;
  }

  // Users Tab

  const usersTableColumns = [
    { key: 'displayName', label: 'Name', sortable: true, sort: stringSortCompare },
    { key: 'actions', label: 'Lab Access' },
  ];

  const usersTableItems = computed(() => {
    let filteredLabUsers = labUsers.value;

    if (searchOutput.value.trim()) {
      filteredLabUsers = labUsers.value.filter((labUser: LabUser) => {
        const searchString = `${labUser.displayName} ${labUser.UserEmail}`.toLowerCase();
        return searchString.includes(searchOutput.value.toLowerCase());
      });
    }

    return filteredLabUsers.sort((userA, userB) => {
      // Lab Manager users first
      if (userA.LabManager && !userB.LabManager) return -1;
      if (!userA.LabManager && userB.LabManager) return 1;
      // then Lab Technicians
      if (userA.LabTechnician && !userB.LabTechnician) return -1;
      if (!userA.LabTechnician && userB.LabTechnician) return 1;
      // then sort by name
      return stringSortCompare(userA.displayName, userB.displayName);
    });
  });

  function showRemoveUserDialog(user: LabUser) {
    userToRemove.value = user;
    primaryMessage.value = `Are you sure you want to remove ${user.displayName} from ${labName.value}?`;
    isOpen.value = true;
  }

  // Seqera Pipelines Tab

  const seqeraPipelinesTableColumns = [
    { key: 'Name', label: 'Name' },
    { key: 'description', label: 'Description' },
    { key: 'actions', label: 'Actions' },
  ];

  const seqeraPipelinesActionItems = (pipeline: any) => [
    [{ label: 'Run', click: () => viewRunSeqeraPipeline(pipeline) }],
  ];

  function viewRunSeqeraPipeline(pipeline: SeqeraPipeline) {
    $router.push({
      path: `/labs/${props.labId}/run-pipeline/${pipeline.pipelineId}`,
      query: {
        seqeraRunTempId: uuidv4(),
      },
    });
  }

  // Omics Workflows Tab

  const omicsWorkflowsTableColumns = [
    { key: 'Name', label: 'Name' },
    { key: 'description', label: 'Description' },
    { key: 'actions', label: 'Actions' },
  ];

  const omicsWorkflowsActionItems = (workflow: any) => [
    [{ label: 'Run', click: () => viewRunOmicsWorkflow(workflow) }],
  ];

  function viewRunOmicsWorkflow(workflow: OmicsWorkflow) {
    $router.push({
      path: `/labs/${props.labId}/run-workflow/${workflow.id}`,
      query: {
        omicsRunTempId: uuidv4(),
      },
    });
  }

  // the rest

  function showRedirectModal() {
    isMissingPATModalOpen.value = true;
  }

  function switchToSettingsTab() {
    isMissingPATModalOpen.value = false;
    tabIndex.value = tabItems.value.findIndex((tab) => tab.key === 'details');
    $router.push({ query: { ...$router.currentRoute.query, tab: tabItems.value[tabIndex.value].label } });
  }

  async function handleRemoveUserFromLab() {
    let maybeDisplayName = 'user';
    try {
      isOpen.value = false;
      useUiStore().setRequestPending('removeUserFromLab');
      const { displayName, UserId } = userToRemove.value;
      maybeDisplayName = displayName;

      const res: DeletedResponse = await $api.labs.removeUser(props.labId, UserId);

      if (res?.Status !== 'Success') {
        throw new Error(`Failed to remove ${displayName} from ${labName.value}`);
      }

      useToastStore().success(`Successfully removed ${displayName} from ${labName.value}`);
    } catch (error) {
      useToastStore().error(`Failed to remove ${maybeDisplayName} from ${labName.value}`);
    } finally {
      await getLabUsers();
      userToRemove.value = undefined;
      useUiStore().setRequestComplete('removeUserFromLab');
    }
  }

  async function handleAssignLabRole({ user, role }: { user: LabUser; role: LaboratoryRolesEnum }) {
    const { displayName, UserId } = user;
    const isLabManager = role === LaboratoryRolesEnumSchema.enum.LabManager;

    try {
      useUiStore().setRequestPending('assignLabRole');

      const res: EditUserResponse = await $api.labs.editUserLabAccess(props.labId, UserId, isLabManager);

      if (res?.Status !== 'Success') {
        throw new Error(`Failed to assign the ${role} role to ${displayName} in ${labName.value}`);
      }

      useToastStore().success(`Successfully assigned the ${role} role to ${displayName} in ${labName.value}`);
    } catch (error) {
      useToastStore().error(`Failed to assign the ${role} role to ${displayName} in ${labName.value}`);
    } finally {
      await getLabUsers();
      useUiStore().setRequestComplete('assignLabRole');
    }
  }

  function getAssignedLabRole(labUserDetails: LaboratoryUserDetails): LaboratoryRolesEnum {
    if (labUserDetails.LabManager) {
      return LaboratoryRolesEnumSchema.enum.LabManager;
    }
    return LaboratoryRolesEnumSchema.enum.LabTechnician;
  }

  function getLabUser(labUserDetails: LaboratoryUserDetails, labUsers: LaboratoryUser[]): LabUser {
    const labUser = labUsers.find((labUser) => labUser.UserId === labUserDetails.UserId);
    if (!labUser) {
      throw new Error(`Lab user not found for user ID: ${labUserDetails.UserId}`);
    }
    const assignedRole = getAssignedLabRole(labUserDetails);

    const displayName = useUser().displayName({
      preferredName: labUserDetails.PreferredName ?? null,
      firstName: labUserDetails.FirstName ?? null,
      lastName: labUserDetails.LastName ?? null,
      email: labUserDetails.UserEmail,
    });

    return {
      ...labUserDetails,
      status: labUser.Status,
      assignedRole,
      displayName,
    } as LabUser;
  }

  async function getLabUsers(): Promise<void> {
    useUiStore().setRequestPending('getLabUsers');
    try {
      const _labUsersDetails: LaboratoryUserDetails[] = await $api.labs.usersDetails(props.labId);
      const _labUsers: LaboratoryUser[] = await $api.labs.listLabUsersByLabId(props.labId);
      labUsers.value = _labUsersDetails.map((user) => getLabUser(user, _labUsers));
    } catch (error) {
      console.error('Error retrieving lab users', error);
      useToastStore().error('Failed to retrieve lab users');
    } finally {
      useUiStore().setRequestComplete('getLabUsers');
    }
  }

  async function loadLabData(): Promise<void> {
    useUiStore().setRequestPending('loadLabData');
    try {
      await labStore.loadLab(props.labId);
    } catch (error) {
      console.error('Error retrieving Lab data', error);
    } finally {
      useUiStore().setRequestComplete('loadLabData');
    }
  }

  async function pollFetchLaboratoryRuns() {
    runsTableRefreshKey.value++;
    await requestLabRunStatusCheck();
    intervalId = window.setTimeout(pollFetchLaboratoryRuns, 2 * 60 * 1000);
  }

  async function requestLabRunStatusCheck() {
    const TERMINAL_STATUSES = ['FAILED', 'SUCCEEDED', 'CANCELLED', 'COMPLETED', 'DELETED'];
    try {
      const nonTerminalRunIds = runsTableItems.value
        .filter((run) => !TERMINAL_STATUSES.includes(run.Status))
        .map((run) => run.RunId);

      if (nonTerminalRunIds.length > 0) {
        await $api.labs.requestLabRunStatusCheck(props.labId, nonTerminalRunIds);
      }
    } catch (error) {
      console.error('Failed to request lab run status check', error);
    }
  }

  async function getSeqeraPipelines(): Promise<void> {
    useUiStore().setRequestPending('getSeqeraPipelines');
    try {
      await seqeraPipelinesStore.loadPipelinesForLab(props.labId);
    } catch (error) {
      console.error('Error retrieving pipelines', error);
    } finally {
      useUiStore().setRequestComplete('getSeqeraPipelines');
    }
  }

  async function getOmicsWorkflows(): Promise<void> {
    useUiStore().setRequestPending('getOmicsWorkflows');
    try {
      await omicsWorkflowsStore.loadWorkflowsForLab(props.labId);
    } catch (error) {
      console.error('Error retrieving pipelines', error);
    } finally {
      useUiStore().setRequestComplete('getOmicsWorkflows');
    }
  }

  // this anticipates these store values being needed on run click
  async function getSeqeraRuns(): Promise<void> {
    useUiStore().setRequestPending('getSeqeraRuns');
    try {
      await runStore.loadSeqeraRunsForLab(props.labId);
    } catch (error) {
      console.error('Error retrieving Seqera runs', error);
    } finally {
      useUiStore().setRequestComplete('getSeqeraRuns');
    }
  }

  // this anticipates these store values being needed on run click
  async function getOmicsRuns(): Promise<void> {
    useUiStore().setRequestPending('getOmicsRuns');
    try {
      await runStore.loadOmicsRunsForLab(props.labId);
    } catch (error) {
      console.error('Error retrieving Omics runs', error);
    } finally {
      useUiStore().setRequestComplete('getOmicsRuns');
    }
  }

  function updateSearchOutput(newVal: any) {
    searchOutput.value = newVal;
  }

  async function handleUserAddedToLab() {
    showAddUserModule.value = false;
    await getLabUsers();
  }

  async function handleCancelDialogAction() {
    const runId = runToCancel.value?.RunId;
    const runName = runToCancel.value?.RunName;
    const runPlatform = runToCancel.value?.Platform;

    if (!runId || !runName || !runPlatform) {
      throw new Error('runToCancel is missing required information');
    }

    try {
      if (runPlatform === 'Seqera Cloud') {
        uiStore.setRequestPending('cancelSeqeraRun');
        await $api.seqeraRuns.cancelPipelineRun(props.labId, runId);
      } else {
        uiStore.setRequestPending('cancelOmicsRun');
        await $api.omicsRuns.cancelWorkflowRun(props.labId, runId);
      }
    } catch (e) {
      useToastStore().error('Failed to cancel run');
    }

    isCancelDialogOpen.value = false;
    uiStore.setRequestComplete('cancelSeqeraRun');
    uiStore.setRequestComplete('cancelOmicsRun');

    await getSeqeraRuns();
    await getOmicsRuns();
  }

  async function handleDetailsUpdated() {
    await loadLabData();

    // the tabs can change after details are updated from adding/removing a compute integration, so update tab index
    setTimeout(setTabIndex, 100); // there's a slight delay to get around a race condition
  }

  watch(lab, async (lab) => {
    if (lab === null) {
      return;
    }

    const promises = [getLabUsers()];

    if (props.superuser) {
      // superuser doesn't view pipelines/workflows or runs so just fetch the users
      await Promise.all(promises);
      return;
    }

    if (lab.NextFlowTowerEnabled) {
      if (lab.HasNextFlowTowerAccessToken == null) {
        // Current lab doesn't have the correct details
        if (uiStore.isRequestPending('loadLabData')) {
          // In the process of loading the lab, which will trigger this code again when it completes
        } else {
          loadLabData(); // Refresh the lab
        }
      } else if (!lab.HasNextFlowTowerAccessToken) {
        // Seqera enabled but creds not present, show the modal
        missingPAT.value = true;
        showRedirectModal();
      } else {
        // fetch the Seqera stuff
        missingPAT.value = false;
        promises.push(getSeqeraPipelines());
        promises.push(getSeqeraRuns());
      }
    }

    if (lab.AwsHealthOmicsEnabled) {
      // fetch the Omics stuff
      promises.push(getOmicsWorkflows());
      promises.push(getOmicsRuns());
    }

    await Promise.all(promises);
  });

  // Note: the UTabs :ui attribute has to be defined locally in this file - if it is imported from another file,
  //  Tailwind won't pick up and include the classes used and styles will be missing.
  // To keep the tab styling consistent throughout the app, any changes made here need to be duplicated to all other
  //  UTabs that use an "EGTabsStyles" as input to the :ui attribute.
  const EGTabsStyles = {
    base: 'focus:outline-none',
    list: {
      base: '!flex rounded-none mb-6 mt-0',
      padding: 'p-0',
      height: 'h-14',
      marker: {
        background: '',
        shadow: '',
      },
      tab: {
        base: 'font-serif w-auto mr-3 rounded-xl border border-solid',
        background: '',
        active: 'text-white bg-primary border-primary',
        inactive: 'font-serif text-text-body border-background-dark-grey',
        height: '',
        padding: 'px-5 py-2',
        size: 'text-sm',
      },
    },
  };
</script>

<template>
  <EGPageHeader
    :title="labName"
    description="View your Lab users, details and pipelines/workflows"
    :back-action="() => (superuser ? $router.push(`/orgs/${orgId || ''}`) : $router.push('/labs'))"
    :show-back="true"
    show-org-breadcrumb
    show-lab-breadcrumb
  >
    <EGButton
      label="Add Lab Users"
      v-if="!superuser"
      :disabled="!canAddUsers"
      @click="showAddUserModule = !showAddUserModule"
    />
    <EGAddLabUsersModule
      v-if="showAddUserModule && !!orgId"
      @added-user-to-lab="handleUserAddedToLab()"
      :org-id="orgId"
      :lab-id="labId"
      :lab-name="labName"
      :lab-users="labUsers"
      class="mt-2"
    />
  </EGPageHeader>

  <UTabs
    v-if="lab"
    :ui="EGTabsStyles"
    :default-index="0"
    :items="tabItems"
    :model-value="tabIndex"
    @update:model-value="
      (newIndex) => {
        $router.push({ query: { ...$router.currentRoute.query, tab: tabItems[newIndex].label } });
        tabIndex = newIndex;
      }
    "
  >
    <template #item="{ item }">
      <!-- Runs tab -->
      <div v-if="item.key === 'runs'">
        <div class="mb-6 flex flex-row items-center gap-4">
          <UCheckbox label="My runs only" :ui="{ base: 'size-[24px]' }" v-model="runsTableFilterMyRunsOnly" />
        </div>

        <EGTable
          :row-click-action="viewRunDetails"
          :table-data="runsTableItems"
          :columns="runsTableColumns"
          v-model:sort="runsTableSort"
          :is-loading="useUiStore().anyRequestPending(['loadLabData', 'loadLabRuns'])"
          :show-pagination="!useUiStore().anyRequestPending(['loadLabData', 'loadLabRuns'])"
        >
          <template #RunName-data="{ row: run }">
            <div v-if="run.RunName" class="text-body text-sm font-medium">{{ run.RunName }}</div>
            <div v-if="run.WorkflowName" class="text-muted text-xs font-normal">{{ run.WorkflowName }}</div>
          </template>

          <template #CreatedAt-data="{ row: run }">
            <div class="text-body text-sm font-medium">{{ getDate(run.CreatedAt) }}</div>
            <div class="text-muted">{{ getTime(run.CreatedAt) }}</div>
          </template>

          <template #lastUpdated-data="{ row: run }">
            <div class="text-body text-sm font-medium">{{ getDate(run.ModifiedAt) }}</div>
            <div class="text-muted">{{ getTime(run.ModifiedAt) }}</div>
          </template>

          <template #Status-data="{ row: run }">
            <EGStatusChip :status="run.Status" />
          </template>

          <template #Owner-data="{ row: run }">
            <div class="text-body text-sm font-medium">{{ run.Owner }}</div>
          </template>

          <template #actions-data="{ row }">
            <div class="flex justify-end">
              <EGActionButton :items="runsActionItems(row)" class="ml-2" @click="$event.stopPropagation()" />
            </div>
          </template>

          <template #empty-state>
            <div class="text-muted flex h-24 items-center justify-center font-normal">
              There are no Runs in your Lab
            </div>
          </template>
        </EGTable>
      </div>

      <!-- Seqera Pipelines tab -->
      <div v-if="item.key === 'seqeraPipelines'">
        <EGTable
          :row-click-action="viewRunSeqeraPipeline"
          :table-data="seqeraPipelines"
          :columns="seqeraPipelinesTableColumns"
          :is-loading="useUiStore().anyRequestPending(['loadLabData', 'getSeqeraPipelines'])"
          :show-pagination="!useUiStore().anyRequestPending(['loadLabData', 'getSeqeraPipelines'])"
        >
          <template #Name-data="{ row: pipeline }">
            <div class="flex items-center">
              {{ pipeline?.name }}
            </div>
          </template>

          <template #description-data="{ row: pipeline }">
            {{ pipeline?.description }}
          </template>

          <template #actions-data="{ row }">
            <div class="flex justify-end">
              <EGActionButton :items="seqeraPipelinesActionItems(row)" class="ml-2" @click="$event.stopPropagation()" />
            </div>
          </template>

          <template #empty-state>
            <div class="text-muted flex h-24 items-center justify-center font-normal">
              There are no Pipelines assigned to this Lab
            </div>
          </template>
        </EGTable>
      </div>

      <!-- HealthOmics Pipelines tab -->
      <div v-if="item.key === 'omicsWorkflows'">
        <EGTable
          :row-click-action="viewRunOmicsWorkflow"
          :table-data="omicsWorkflows"
          :columns="omicsWorkflowsTableColumns"
          :is-loading="useUiStore().anyRequestPending(['loadLabData', 'getOmicsWorkflows'])"
          :show-pagination="!useUiStore().anyRequestPending(['loadLabData', 'getOmicsWorkflows'])"
        >
          <template #Name-data="{ row: workflow }">
            <div class="flex items-center">
              {{ workflow?.name }}
            </div>
          </template>

          <template #description-data="{ row: workflow }">
            {{ workflow?.description }}
          </template>

          <template #actions-data="{ row: workflow }">
            <div class="flex justify-end">
              <EGActionButton
                :items="omicsWorkflowsActionItems(workflow)"
                class="ml-2"
                @click="$event.stopPropagation()"
              />
            </div>
          </template>

          <template #empty-state>
            <div class="text-muted flex h-24 items-center justify-center font-normal">
              There are no Workflows assigned to this Lab
            </div>
          </template>
        </EGTable>
      </div>

      <!-- Lab Users tab -->
      <div v-if="item.key === 'users'">
        <EGSearchInput
          @input-event="updateSearchOutput"
          placeholder="Search user"
          :disabled="useUiStore().anyRequestPending(['loadLabData', 'getLabUsers', 'addUserToLab'])"
          class="my-6 w-[408px]"
        />

        <EGDialog
          actionLabel="Remove User"
          :actionVariant="ButtonVariantEnum.enum.destructive"
          cancelLabel="Cancel"
          :cancelVariant="ButtonVariantEnum.enum.secondary"
          @action-triggered="handleRemoveUserFromLab"
          :primaryMessage="primaryMessage"
          v-model="isOpen"
        />

        <EGTable
          :table-data="usersTableItems"
          :columns="usersTableColumns"
          :is-loading="useUiStore().anyRequestPending(['loadLabData', 'getLabUsers', 'assignLabRole'])"
          :show-pagination="!useUiStore().anyRequestPending(['loadLabData', 'getLabUsers', 'assignLabRole'])"
        >
          <template #displayName-data="{ row: labUser }">
            <div class="flex items-center">
              <EGUserDisplay :name="labUser.displayName" :email="labUser.UserEmail" />
            </div>
          </template>

          <template #actions-data="{ row: labUser }">
            <div class="flex items-center">
              <EGUserRoleDropdownNew
                :show-remove-from-lab="true"
                :key="labUser?.LabManager"
                :disabled="
                  useUiStore().anyRequestPending(['loadLabData', 'getLabUsers']) ||
                  !userStore.canEditLabUsers(labId) ||
                  userStore.isSuperuser
                "
                :user="labUser"
                @assign-lab-role="handleAssignLabRole($event)"
                @remove-user-from-lab="showRemoveUserDialog($event.user)"
              />
            </div>
          </template>

          <template #empty-state>
            <div class="text-muted flex h-24 items-center justify-center font-normal">
              There are no users in your Lab
            </div>
          </template>
        </EGTable>
      </div>

      <!-- Lab Details -->
      <div v-if="item.key === 'details'">
        <EGFormLabDetails @updated="handleDetailsUpdated" />
      </div>
    </template>
  </UTabs>

  <EGDialog
    action-label="Cancel Run"
    :action-variant="ButtonVariantEnum.enum.destructive"
    @action-triggered="handleCancelDialogAction"
    :primary-message="`Are you sure you would like to cancel ${runToCancel?.RunName}?`"
    secondary-message="This will stop any progress made."
    v-model="isCancelDialogOpen"
    :buttons-disabled="uiStore.anyRequestPending(['cancelSeqeraRun', 'cancelOmicsRun'])"
  />

  <EGDialog
    action-label="Okay"
    :action-variant="ButtonVariantEnum.enum.primary"
    @action-triggered="switchToSettingsTab"
    primary-message="No Personal Access Token found"
    secondary-message="A Personal Access Token is required to run a pipeline. Please click 'Edit' in the next screen to set it."
    v-model="isMissingPATModalOpen"
  />
</template>
