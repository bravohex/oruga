<script setup lang="ts">
import { ref } from "vue";

const radio = ref("Jack");
</script>

<template>
    <section>
        <o-field>
            <o-radio
                v-model="radio"
                name="base"
                native-value="Flint"
                label="Flint" />

            <o-radio
                v-model="radio"
                name="base"
                native-value="Silver"
                label="Silver" />

            <o-radio
                v-model="radio"
                name="base"
                native-value="Jack"
                label="Jack" />

            <o-radio
                v-model="radio"
                name="base"
                native-value="Vane"
                label="Vane"
                disabled />
        </o-field>

        <p><b>Selection:</b> {{ radio }}</p>
    </section>
</template>
