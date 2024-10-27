<template>
  <input v-bind="PROP.directives" />
</template>

<script>
export default {
  name: "CommonPlainInput",
  props: {
    PROP: {
      type: Object,
      required: true,
    },
  },
};
</script>
