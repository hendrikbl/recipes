// code taken from https://github.com/bootstrap-vue/bootstrap-vue/issues/4977#issuecomment-740215609 and modified
<template>
  <b-input-group>
    <b-input-group-prepend>
      <b-button
        variant="outline-primary"
        class="py-1 px-3"
        :size="size"
        @click="valueChange(value - 1)"
      >
        <i class="fas fa-minus" />
      </b-button>
    </b-input-group-prepend>

    <b-input-group-append>
      <b-button
        variant="outline-primary"
        class="py-1 px-3"
        :size="size"
        @click="valueChange(value + 1)"
      >
        <i class="fas fa-plus" />
      </b-button>
    </b-input-group-append>
  </b-input-group>
</template>

<script>
export default {
  name: "CustomInputSpinButton",

  props: {
    size: {
      type: String,
      required: false,
      default: "md",
      validator: function (value) {
        return ["sm", "md", "lg"].includes(value);
      },
    },

    value: {
      type: Number,
      required: true,
    },
  },

  methods: {
    valueChange(newValue) {
      if (newValue <= 0) {
        this.$emit("input", 0);
      } else {
        this.$emit("input", newValue);
      }
    },
  },
};
</script>

<style scoped>
</style>