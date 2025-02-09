<script setup lang="ts">
  import { LocationSummary, LocationUpdate } from "~~/lib/api/types/data-contracts";
  import { useLocationStore } from "~~/stores/locations";

  definePageMeta({
    middleware: ["auth"],
  });

  const route = useRoute();
  const api = useUserApi();
  const toast = useNotifier();

  const locationId = computed<string>(() => route.params.id as string);

  const { data: location } = useAsyncData(locationId.value, async () => {
    const { data, error } = await api.locations.get(locationId.value);
    if (error) {
      toast.error("Failed to load location");
      navigateTo("/home");
      return;
    }

    if (data.parent) {
      parent.value = locations.value.find(l => l.id === data.parent.id);
    }

    return data;
  });

  const confirm = useConfirm();

  async function confirmDelete() {
    const { isCanceled } = await confirm.open(
      "Are you sure you want to delete this location and all of its items? This action cannot be undone."
    );
    if (isCanceled) {
      return;
    }

    const { error } = await api.locations.delete(locationId.value);
    if (error) {
      toast.error("Failed to delete location");
      return;
    }

    toast.success("Location deleted");
    navigateTo("/home");
  }

  const updateModal = ref(false);
  const updating = ref(false);
  const updateData = reactive<LocationUpdate>({
    id: locationId.value,
    name: "",
    description: "",
    parentId: null,
  });

  function openUpdate() {
    updateData.name = location.value?.name || "";
    updateData.description = location.value?.description || "";
    updateModal.value = true;
  }

  async function update() {
    updating.value = true;
    updateData.parentId = parent.value?.id || null;
    const { error, data } = await api.locations.update(locationId.value, updateData);

    if (error) {
      updating.value = false;
      toast.error("Failed to update location");
      return;
    }

    toast.success("Location updated");
    location.value = data;
    updateModal.value = false;
    updating.value = false;
  }

  const locationStore = useLocationStore();
  const locations = computed(() => locationStore.allLocations);

  const parent = ref<LocationSummary | any>({});

  const items = computedAsync(async () => {
    if (!location.value) {
      return [];
    }

    const resp = await api.items.getAll({
      locations: [location.value.id],
    });

    if (resp.error) {
      toast.error("Failed to load items");
      return [];
    }

    return resp.data.items;
  });
</script>

<template>
  <div>
    <!-- Update Dialog -->
    <BaseModal v-model="updateModal">
      <template #title> Update Location </template>
      <form v-if="location" @submit.prevent="update">
        <FormTextField v-model="updateData.name" :autofocus="true" label="Location Name" />
        <FormTextArea v-model="updateData.description" label="Location Description" />
        <LocationSelector v-model="parent" />
        <div class="modal-action">
          <BaseButton type="submit" :loading="updating"> Update </BaseButton>
        </div>
      </form>
    </BaseModal>

    <BaseContainer v-if="location">
      <div class="bg-white rounded p-3">
        <header class="mb-2">
          <div class="flex flex-wrap items-end gap-2">
            <div class="avatar placeholder mb-auto">
              <div class="bg-neutral-focus text-neutral-content rounded-full w-12">
                <Icon name="mdi-package-variant" class="h-7 w-7" />
              </div>
            </div>
            <div>
              <div v-if="location?.parent" class="text-sm breadcrumbs pt-0 pb-0">
                <ul class="text-base-content/70">
                  <li>
                    <NuxtLink :to="`/location/${location.parent.id}`"> {{ location.parent.name }}</NuxtLink>
                  </li>
                  <li>{{ location.name }}</li>
                </ul>
              </div>
              <h1 class="text-2xl pb-1">
                {{ location ? location.name : "" }}
              </h1>
              <div class="flex gap-1 flex-wrap text-xs">
                <div>
                  Created
                  <DateTime :date="location?.createdAt" />
                </div>
              </div>
            </div>
            <div class="ml-auto mt-2 flex flex-wrap items-center justify-between gap-3">
              <div class="btn-group">
                <PageQRCode class="dropdown-left" />
                <BaseButton size="sm" @click="openUpdate">
                  <Icon class="mr-1" name="mdi-pencil" />
                  Edit
                </BaseButton>
              </div>
              <BaseButton class="btn btn-sm" @click="confirmDelete()">
                <Icon name="mdi-delete" class="mr-2" />
                Delete
              </BaseButton>
            </div>
          </div>
        </header>
        <div class="divider my-0 mb-1"></div>
        <Markdown v-if="location && location.description" class="text-base" :source="location.description"> </Markdown>
      </div>
      <section v-if="location && items">
        <ItemViewSelectable :items="items" />
      </section>
    </BaseContainer>
  </div>
</template>
