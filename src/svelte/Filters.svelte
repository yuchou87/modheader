<script>
  import IconButton from "@smui/icon-button";
  import Button, { Label } from "@smui/button";
  import Checkbox from "@smui/checkbox";
  import Menu from "@smui/menu";
  import Select, { Option } from "@smui/select";
  import List, { Item, Separator, Text } from "@smui/list";
  import {
    mdiPlus,
    mdiClose,
    mdiTrashCanOutline,
    mdiHelpCircleOutline,
    mdiSortAlphabeticalAscending,
    mdiSortAlphabeticalDescending,
    mdiDotsVertical
  } from "@mdi/js";
  import lodashUniq from "lodash/uniq";
  import lodashOrderBy from "lodash/orderBy";
  import lodashCloneDeep from "lodash/cloneDeep";
  import lodashDebounce from "lodash/debounce";
  import {
    addFilter,
    removeFilter,
    selectedProfile,
    commitChange
  } from "../js/datasource";
  import {
    DISABLED_COLOR,
    PRIMARY_COLOR,
    MAX_AUTOCOMPLETE_LENGTH
  } from "../js/constants";
  import AutoComplete from "./Autocomplete.svelte";
  import MdiIcon from "./MdiIcon.svelte";
  import ResourceTypeMenu from "./ResourceTypeMenu.svelte";

  const FILTER_TYPES = {
    urls: "URL Pattern",
    excludeUrls: "Exclude URL Pattern",
    types: "Resource Type"
  };

  export let filters;
  let selectedFilter;
  let dialog;
  let sortMenu;
  let clazz;
  let resourceTypeMenuLocation;
  export { clazz as class };

  let allChecked;
  let allUnchecked;

  function expandEditor(filter) {
    selectedFilter = filter;
    dialog.open();
  }

  function sort(field, order) {
    filters = lodashOrderBy(filters, [field], [order]);
    refreshFilters();
  }

  function toggleAll() {
    if (!allChecked) {
      filters.forEach(f => (f.enabled = true));
    } else {
      filters.forEach(f => (f.enabled = false));
    }
    refreshFilters();
  }

  function refreshFilters() {
    commitChange({ filters });
  }

  const refreshFiltersDebounce = lodashDebounce(
    () => {
      refreshFilters();
      allChecked = filters.every(f => f.enabled);
      allUnchecked = filters.every(f => !f.enabled);
    },
    500,
    { leading: true, trailing: true }
  );
  $: filters, refreshFiltersDebounce();
</script>

<style scoped>
  :global(.filter-select) {
    height: 26px;
    width: 170px;
  }

  :global(.filter-select .mdc-select__selected-text) {
    padding: 0;
  }

  :global(.filter-select) :global(.mdc-select__dropdown-icon) {
    top: 0;
    bottom: 0;
  }

  :global(.data-table-value-cell) {
    width: 350px;
  }
</style>

<div class="data-table {clazz}">
  <div class="data-table-row data-table-title-row">
    <Checkbox
      class="data-table-cell flex-fixed-icon"
      bind:checked={allChecked}
      indeterminate={!allChecked && !allUnchecked}
      on:click={toggleAll}
      disabled={$selectedProfile.filters.length === 0} />
    <h3 class="data-table-title data-table-cell flex-grow">Filters</h3>
    <div class="data-table-cell">
      <IconButton
        aria-label="Expand"
        class="medium-icon-button data-table-cell flex-fixed-icon"
        on:click={() => sortMenu.setOpen(true)}>
        <MdiIcon size="32" color="#666" icon={mdiDotsVertical} />
      </IconButton>

      <Menu bind:this={sortMenu} quickOpen>
        <List>
          <Item on:SMUI:action={() => addFilter()}>
            <MdiIcon
              class="more-menu-icon"
              size="24"
              icon={mdiPlus}
              color="#666" />
            <Text>Add</Text>
          </Item>
          <Item on:SMUI:action={() => commitChange({ filters: [] })}>
            <MdiIcon
              class="more-menu-icon"
              size="24"
              icon={mdiTrashCanOutline}
              color="#666" />
            <Text>Clear all</Text>
          </Item>
          <Separator nav />
          <Item on:SMUI:action={() => sort('type', 'asc')}>
            <MdiIcon
              class="more-menu-icon"
              size="24"
              icon={mdiSortAlphabeticalAscending}
              color="#666" />
            <Text>Type - ascending</Text>
          </Item>
          <Item on:SMUI:action={() => sort('type', 'desc')}>
            <MdiIcon
              class="more-menu-icon"
              size="24"
              icon={mdiSortAlphabeticalDescending}
              color="#666" />
            <Text>Type - descending</Text>
          </Item>
          <Item on:SMUI:action={() => sort('urlRegex', 'asc')}>
            <MdiIcon
              class="more-menu-icon"
              size="24"
              icon={mdiSortAlphabeticalAscending}
              color="#666" />
            <Text>URL regex - ascending</Text>
          </Item>
          <Item on:SMUI:action={() => sort('urlRegex', 'desc')}>
            <MdiIcon
              class="more-menu-icon"
              size="24"
              icon={mdiSortAlphabeticalDescending}
              color="#666" />
            <Text>URL regex - descending</Text>
          </Item>
          {#if !$selectedProfile.hideComment}
            <Item on:SMUI:action={() => sort('comment', 'asc')}>
              <MdiIcon
                class="more-menu-icon"
                size="24"
                icon={mdiSortAlphabeticalAscending}
                color="#666" />
              <Text>Comment - ascending</Text>
            </Item>
            <Item on:SMUI:action={() => sort('comment', 'desc')}>
              <MdiIcon
                class="more-menu-icon"
                size="24"
                icon={mdiSortAlphabeticalDescending}
                color="#666" />
              <Text>Comment - descending</Text>
            </Item>
          {/if}
        </List>
      </Menu>
    </div>
  </div>
  {#each filters as filter, filterIndex}
    <div
      class="data-table-row {filter.enabled ? '' : 'data-table-row-unchecked'}">
      <Checkbox
        bind:checked={filter.enabled}
        indeterminate={false}
        on:change={refreshFilters}
        class="data-table-cell flex-fixed-icon" />
      <Select
        bind:value={filter.type}
        noLabel
        enhanced
        class="data-table-cell filter-select"
        input$class="filter-select-field">
        {#each Object.entries(FILTER_TYPES) as [value, label]}
          <Option {value} selected={filter.type === value}>{label}</Option>
        {/each}
      </Select>
      {#if filter.type === 'urls' || filter.type === 'excludeUrls'}
        <AutoComplete
          bind:value={filter.urlRegex}
          placeholder=".*://.*.google.com/.*" />
      {:else}
        <ResourceTypeMenu
          bind:resourceType={filter.resourceType}
          {resourceTypeMenuLocation} />
      {/if}
      {#if !$selectedProfile.hideComment}
        <AutoComplete bind:value={filter.comment} placeholder="Comment" />
      {/if}
      <IconButton
        dense
        aria-label="Delete"
        class="small-icon-button data-table-cell flex-fixed-icon"
        on:click={() => removeFilter(filterIndex)}>
        <MdiIcon size="24" icon={mdiClose} color="red" />
      </IconButton>
    </div>
  {/each}
</div>
