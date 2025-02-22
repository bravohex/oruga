<script setup lang="ts" generic="T">
import { useId, computed } from "vue";

import { getDefault } from "@/utils/config";
import { isDefined, isEqual } from "@/utils/helpers";
import { defineClasses, useProviderChild } from "@/composables";

import type { DropdownComponent } from "./types";
import type { DropdownItemProps } from "./props";

/**
 * @displayName Dropdown Item
 */
defineOptions({
    isOruga: true,
    name: "ODropdownItem",
    configField: "dropdown",
});

const props = withDefaults(defineProps<DropdownItemProps<T>>(), {
    override: undefined,
    value: undefined,
    label: undefined,
    disabled: false,
    clickable: true,
    tag: () => getDefault("dropdown.itemTag", "div"),
    tabindex: 0,
    ariaRole: () => getDefault("dropdown.itemAriaRole", "listitem"),
});

const itemValue = props.value || useId();

const emits = defineEmits<{
    /**
     * onclick event
     * @param value {string | number | object} value prop data
     * @param event {event} Native Event
     */
    click: [value: T, event: Event];
}>();

/** inject functionalities and data from the parent component */
const { parent, item } = useProviderChild<DropdownComponent<T>>();

const isClickable = computed(
    () => !parent.value.disabled && !props.disabled && props.clickable,
);

const isActive = computed(() => {
    if (!isDefined(parent.value.selected)) return false;
    if (parent.value.multiple && Array.isArray(parent.value.selected))
        return parent.value.selected.some((selected: T) =>
            isEqual(itemValue, selected),
        );
    return isEqual(itemValue, parent.value.selected);
});

/** Click listener, select the item. */
function selectItem(event: Event): void {
    if (!isClickable.value) return;
    parent.value.selectItem(itemValue as T);
    emits("click", itemValue as T, event);
}

// --- Computed Component Classes ---

const rootClasses = defineClasses(
    ["itemClass", "o-drop__item"],
    [
        "itemDisabledClass",
        "o-drop__item--disabled",
        null,
        computed(() => parent.value.disabled || props.disabled),
    ],
    ["itemActiveClass", "o-drop__item--active", null, isActive],
    ["itemClickableClass", "o-drop__item--clickable", null, isClickable],
);
</script>

<template>
    <component
        :is="tag"
        :class="rootClasses"
        data-oruga="dropdown-item"
        :data-id="`dropdown-${item.identifier}`"
        :role="ariaRole"
        :tabindex="tabindex"
        :aria-selected="isActive"
        :aria-disabled="disabled"
        @click="selectItem"
        @keypress.enter="selectItem">
        <!--
            @slot Override the label, default is label prop 
        -->
        <slot>{{ label }}</slot>
    </component>
</template>
