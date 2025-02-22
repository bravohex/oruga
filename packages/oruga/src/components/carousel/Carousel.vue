<script setup lang="ts">
import {
    computed,
    watch,
    onBeforeUnmount,
    onMounted,
    ref,
    nextTick,
    readonly,
    toRaw,
    useTemplateRef,
} from "vue";

import OIcon from "../icon/Icon.vue";

import { getDefault } from "@/utils/config";
import { sign, mod, bound, isDefined } from "@/utils/helpers";
import { isClient } from "@/utils/ssr";
import {
    defineClasses,
    useEventListener,
    useProviderParent,
} from "@/composables";

import type { CarouselComponent } from "./types";
import type { ClassBind } from "@/types";
import type { CarouselProps } from "./props";

/**
 * A Slideshow for cycling images in confined spaces
 * @displayName Carousel
 * @requires ./CarouselItem.vue
 * @style _carousel.scss
 */
defineOptions({
    isOruga: true,
    name: "OCarousel",
    configField: "carousel",
});

const props = withDefaults(defineProps<CarouselProps>(), {
    override: undefined,
    modelValue: 0,
    dragable: true,
    interval: () => getDefault("carousel.interval", 3500),
    autoplay: false,
    pauseHover: false,
    repeat: false,
    overlay: false,
    indicators: true,
    indicatorInside: false,
    indicatorMode: "click",
    indicatorPosition: () => getDefault("carousel.indicatorPosition", "bottom"),
    indicatorStyle: () => getDefault("carousel.indicatorStyle", "dots"),
    itemsToShow: () => getDefault("carousel.itemsToShow", 1),
    itemsToList: () => getDefault("carousel.itemsToList", 1),
    arrows: () => getDefault("carousel.arrows", true),
    arrowsHover: () => getDefault("carousel.arrowsHover", true),
    iconPack: () => getDefault("carousel.iconPack"),
    iconSize: () => getDefault("carousel.iconSize"),
    iconPrev: () => getDefault("carousel.iconPrev", "chevron-left"),
    iconNext: () => getDefault("carousel.iconNext", "chevron-right"),
    breakpoints: () => ({}),
});

const emits = defineEmits<{
    /**
     * modelValue prop two-way binding
     * @param value {number} updated modelValue prop
     */
    "update:model-value": [value: number];
    /**
     * on carousel scroll event
     * @param value {number} scroll index
     */
    scroll: [value: number];
    /**
     * on item click event
     * @param event {event} native event
     */
    click: [event: Event];
}>();

const rootRef = useTemplateRef("rootElement");

function restartTimer(): void {
    pauseTimer();
    startTimer();
}

// provided data is a computed ref to enjure reactivity
const provideData = computed<CarouselComponent>(() => ({
    restartTimer,
    itemWidth: itemWidth.value,
    activeIndex: scrollIndex.value,
    onClick: (event: Event): void => emits("click", event),
    setActive: (index: number): void => switchTo(index),
}));

/** provide functionalities and data to child item components */
const { childItems } = useProviderParent({ rootRef, data: provideData });

const activeIndex = defineModel<number>({ default: 0 });
const scrollIndex = ref(props.modelValue);

let resizeObserver: ResizeObserver | undefined;
const windowWidth = ref(0);

if (isClient && window.ResizeObserver) {
    resizeObserver = new window.ResizeObserver(onRefresh);
}

const refresh_ = ref(0);

/** When v-model is changed switch to the new active item. */
watch(
    () => props.modelValue,
    (value) => {
        if (value <= childItems.value.length - 1)
            switchTo(value * settings.value.itemsToList, true);
    },
);

watch([() => props.itemsToList, () => props.itemsToShow], () => onRefresh());

onMounted(() => {
    if (isClient) {
        if (window.ResizeObserver && resizeObserver && rootRef.value)
            resizeObserver.observe(rootRef.value);

        onResized();
        startTimer();
    }
});

onBeforeUnmount(() => {
    if (isClient) {
        if (window.ResizeObserver && resizeObserver)
            resizeObserver.disconnect();

        dragEnd();
        pauseTimer();
    }
});

// add dom event handler
if (isClient) {
    useEventListener(window, "resize", onResized);
    useEventListener(document, "animationend", onRefresh);
    useEventListener(document, "transitionend", onRefresh);
    useEventListener(document, "transitionstart", onRefresh);
}

function onResized(): void {
    windowWidth.value = window.innerWidth;
}

function onRefresh(): void {
    nextTick(() => refresh_.value++);
}

const settings = computed<typeof props>(() => {
    const breakpoints = Object.keys(props.breakpoints)
        .map(Number)
        .sort((a, b) => b - a);

    const breakpoint = breakpoints.filter(
        (breakpoint) => windowWidth.value >= breakpoint,
    )[0];

    const settings = toRaw(
        breakpoint ? { ...props, ...props.breakpoints[breakpoint] } : props,
    );

    // prevent empty values
    if (!settings.itemsToList) settings.itemsToList = 1;
    if (!settings.itemsToShow) settings.itemsToShow = 1;
    return readonly(settings);
});

const itemWidth = computed(() => {
    // Ensure component is mounted
    if (!windowWidth.value || !rootRef.value) return 0;

    // eslint-disable-next-line @typescript-eslint/no-unused-vars
    const r = refresh_.value; // We force the computed property to refresh if this ref is changed

    const rect = rootRef.value.getBoundingClientRect();
    return rect.width / settings.value.itemsToShow;
});

const translation = computed(
    () =>
        -bound(
            delta.value + scrollIndex.value * itemWidth.value,
            0,
            (childItems.value.length - settings.value.itemsToShow) *
                itemWidth.value,
        ),
);

const total = computed(() => childItems.value.length);

const indicatorCount = computed(() =>
    Math.ceil(total.value / settings.value.itemsToList),
);

const indicatorIndex = computed(() =>
    Math.ceil(scrollIndex.value / settings.value.itemsToList),
);

const hasArrows = computed(
    () =>
        (settings.value.arrowsHover && isHovered.value) ||
        !settings.value.arrowsHover,
);

const hasPrev = computed(
    () => (settings.value.repeat || scrollIndex.value > 0) && hasArrows.value,
);

function onPrev(): void {
    switchTo(scrollIndex.value - settings.value.itemsToList);
}

const hasNext = computed(
    () =>
        (settings.value.repeat || scrollIndex.value < total.value - 1) &&
        hasArrows.value,
);

function onNext(): void {
    switchTo(scrollIndex.value + settings.value.itemsToList);
}

function switchTo(index: number, onlyMove?: boolean): void {
    if (settings.value.repeat) index = mod(index, total.value);

    index = bound(index, 0, total.value);
    scrollIndex.value = index;
    emits("scroll", indicatorIndex.value);

    if (!onlyMove)
        activeIndex.value = Math.ceil(index / settings.value.itemsToList);
}

function onModeChange(trigger: string, index: number): void {
    if (props.indicatorMode === trigger)
        switchTo(index * settings.value.itemsToList);
}

// --- Autoplay Feature ---

const isHovered = ref(false);
let timer: NodeJS.Timeout | undefined;

function onMouseEnter(): void {
    isHovered.value = true;
    checkPause();
}

function onMouseLeave(): void {
    isHovered.value = false;
    startTimer();
}

/** When autoplay is changed, start or pause timer accordingly */
watch(
    () => props.autoplay,
    (status) => {
        if (status) startTimer();
        else pauseTimer();
    },
);

/** Since the timer can get paused at the end, if repeat is changed we need to restart it */
watch(
    () => props.repeat,
    (status) => {
        if (status) startTimer();
    },
);

function startTimer(): void {
    if (!props.autoplay || timer) return;
    timer = setInterval(() => {
        if (!props.repeat && !hasNext.value) pauseTimer();
        else onNext();
    }, props.interval);
}

function pauseTimer(): void {
    if (timer) {
        clearInterval(timer);
        timer = undefined;
    }
}

function checkPause(): void {
    if (props.pauseHover && props.autoplay) pauseTimer();
}

// --- Drag & Drop Feature ---

const isTouch = ref(false);
const dragX = ref();
const hold = ref(0);
const delta = ref(0);

const isDragging = computed(() => isDefined(dragX.value));

/** handle drag event */
function onDragStart(event: TouchEvent | MouseEvent): void {
    if (
        isDragging.value ||
        !settings.value.dragable ||
        ((event as MouseEvent).button !== 0 && event.type !== "touchstart")
    )
        return;
    hold.value = Date.now();
    isTouch.value = !!(event as TouchEvent).touches;
    dragX.value = isTouch.value
        ? (event as TouchEvent).touches[0].clientX
        : (event as MouseEvent).clientX;
    if (isTouch.value) {
        pauseTimer();
    }
    if (isClient) {
        window.addEventListener(
            isTouch.value ? "touchmove" : "mousemove",
            dragMove,
        );
        window.addEventListener(
            isTouch.value ? "touchend" : "mouseup",
            dragEnd,
        );
    }
}

function dragMove(event: TouchEvent | MouseEvent): void {
    if (!isDragging.value) return;
    const dragEndX = (event as TouchEvent).touches
        ? (
              (event as TouchEvent).changedTouches[0] ||
              (event as TouchEvent).touches[0]
          ).clientX
        : (event as MouseEvent).clientX;
    delta.value = dragX.value - dragEndX;
    // prevent event if not touch event
    if (!(event as TouchEvent).touches) event.preventDefault();
}

function dragEnd(event?: TouchEvent | MouseEvent): void {
    if (!isDragging.value && !hold.value) return;
    if (hold.value) {
        const signCheck = sign(delta.value);
        const results = Math.round(
            Math.abs(delta.value / itemWidth.value) + 0.15,
        ); // Hack
        switchTo(scrollIndex.value + signCheck * results);
    }
    delta.value = 0;
    dragX.value = undefined;
    if ((event as TouchEvent)?.touches) startTimer();

    if (isClient) {
        window.removeEventListener(
            isTouch.value ? "touchmove" : "mousemove",
            dragMove,
        );
        window.removeEventListener(
            isTouch.value ? "touchend" : "mouseup",
            dragEnd,
        );
    }
}

// --- Computed Component Classes ---

const rootClasses = defineClasses(
    ["rootClass", "o-car"],
    ["overlayClass", "o-car__overlay", null, computed(() => props.overlay)],
);

const wrapperClasses = defineClasses(["wrapperClass", "o-car__wrapper"]);

const itemsClasses = defineClasses(
    ["itemsClass", "o-car__items"],
    ["itemsDraggingClass", "o-car__items--dragging", null, isDragging],
);

const arrowIconClasses = defineClasses([
    "arrowIconClass",
    "o-car__arrow__icon",
]);

const arrowIconPrevClasses = defineClasses([
    "arrowIconPrevClass",
    "o-car__arrow__icon-prev",
]);

const arrowIconNextClasses = defineClasses([
    "arrowIconNextClass",
    "o-car__arrow__icon-next",
]);

const indicatorsClasses = defineClasses(
    ["indicatorsClass", "o-car__indicators"],
    [
        "indicatorsInsideClass",
        "o-car__indicators--inside",
        null,
        computed(() => !!props.indicatorInside),
    ],
    [
        "indicatorsInsidePositionClass",
        "o-car__indicators--inside--",
        computed(() => props.indicatorPosition),
        computed(() => props.indicatorInside && !!props.indicatorPosition),
    ],
);

const indicatorClasses = defineClasses(["indicatorClass", "o-car__indicator"]);

const indicatorItemClasses = defineClasses(
    ["indicatorItemClass", "o-car__indicator__item"],
    [
        "indicatorItemStyleClass",
        "o-car__indicator__item--",
        props.indicatorStyle,
        !!props.indicatorStyle,
    ],
);

const indicatorItemActiveClasses = defineClasses([
    "indicatorItemActiveClass",
    "o-car__indicator__item--active",
]);

function indicatorItemAppliedClasses(index: number): ClassBind[] {
    const activeClasses =
        indicatorIndex.value === index ? indicatorItemActiveClasses.value : [];

    return [...indicatorItemClasses.value, ...activeClasses];
}
</script>

<template>
    <div
        ref="rootElement"
        :class="rootClasses"
        data-oruga="carousel"
        role="region"
        @mouseover="onMouseEnter"
        @mouseleave="onMouseLeave"
        @focus="onMouseEnter"
        @blur="onMouseLeave"
        @keydown.left="onPrev"
        @keydown.right="onNext">
        <div :class="wrapperClasses">
            <div
                :class="itemsClasses"
                :style="'transform:translateX(' + translation + 'px)'"
                tabindex="0"
                role="group"
                draggable="true"
                aria-roledescription="carousel"
                @mousedown="onDragStart"
                @touchstart="onDragStart">
                <!--
                    @slot Display carousel item
                -->
                <slot />
            </div>

            <!--
                @slot Override the arrows
                @binding {boolean} has-prev has prev arrow button 
                @binding {boolean} has-next has next arrow button 
                @binding {(): void} prev switch to prev item function
                @binding {(): void} next switch to next item function
            -->
            <slot
                name="arrow"
                :has-prev="hasPrev"
                :prev="onPrev"
                :has-next="hasNext"
                :next="onNext">
                <template v-if="arrows">
                    <o-icon
                        v-show="hasPrev"
                        :class="[...arrowIconClasses, ...arrowIconPrevClasses]"
                        :pack="iconPack"
                        :icon="iconPrev"
                        :size="iconSize"
                        both
                        role="button"
                        tabindex="0"
                        @click="onPrev"
                        @keydown.enter="onPrev" />
                    <o-icon
                        v-show="hasNext"
                        :class="[...arrowIconClasses, ...arrowIconNextClasses]"
                        :pack="iconPack"
                        :icon="iconNext"
                        :size="iconSize"
                        both
                        role="button"
                        tabindex="0"
                        @click="onNext"
                        @keydown.enter="onNext" />
                </template>
            </slot>
        </div>

        <!--
            @slot Override the indicators
            @binding {number} active active index 
            @binding {(idx: number): void} switch-to switch to item function
            @binding {number} indicator-index current indicator index
        -->
        <slot
            :active="activeIndex"
            :switch-to="switchTo"
            :indicator-index="indicatorIndex"
            name="indicators">
            <template v-if="childItems.length">
                <div v-if="indicators" :class="indicatorsClasses" role="group">
                    <div
                        v-for="(_, index) in indicatorCount"
                        :key="index"
                        :class="indicatorClasses"
                        role="button"
                        tabindex="0"
                        @focus="onModeChange('hover', index)"
                        @mouseover="onModeChange('hover', index)"
                        @click="onModeChange('click', index)"
                        @keypress.enter="onModeChange('click', index)">
                        <!--
                            @slot Override the indicator elements
                            @binding {index} index indicator index 
                        -->
                        <slot :index="index" name="indicator">
                            <span :class="indicatorItemAppliedClasses(index)" />
                        </slot>
                    </div>
                </div>
            </template>
        </slot>

        <template v-if="overlay">
            <!-- @slot Overlay element -->
            <slot name="overlay" />
        </template>
    </div>
</template>
