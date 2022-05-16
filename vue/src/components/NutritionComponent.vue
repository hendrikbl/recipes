<template>
  <div v-if="recipe.nutrition !== null">
    <div class="card">
      <div class="card-body">
        <div class="row">
          <div
            v-for="col in columns"
            :key="col.label"
            class="col-6 col-md-3 mb-3"
          >
            <div class="d-flex flex-column justify-content-center text-center">
              <b class="text-primary">{{ col.value }}</b>
              {{ $t(col.label) }}
            </div>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>

<script>
import { calculateAmount, calculateEnergy, energyHeading } from "@/utils/utils";

export default {
  name: "NutritionComponent",
  props: {
    recipe: Object,
    ingredient_factor: Number,
  },
  computed: {
    columns() {
      const { calories, carbohydrates, fats, proteins } = this.recipe.nutrition;
      return [
        {
          value: calculateEnergy(calories, this.ingredient_factor),
          label: energyHeading(),
        },
        {
          value: calculateAmount(carbohydrates, this.ingredient_factor),
          label: "Carbohydrates",
        },
        {
          value: calculateAmount(fats, this.ingredient_factor),
          label: "Fats",
        },
        {
          value: calculateAmount(proteins, this.ingredient_factor),
          label: "Proteins",
        },
      ];
    },
  },
};
</script>
