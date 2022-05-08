<template>
  <div id="app">
    <template v-if="loading">
      <loading-spinner />
    </template>

    <div v-if="!loading">
      <recipe-switcher
        ref="ref_recipe_switcher"
        @switch="quickSwitch($event)"
      />

      <div class="row mb-4">
        <div class="col-12" style="height: 30vh">
          <recipe-context-menu
            :recipe="recipe"
            :servings="servings"
            class="my-2"
            style="z-index: 300; position: absolute; right: 0"
          />

          <img
            class="recipe-header-image"
            :src="recipe.image"
            :alt="$t('Recipe_Image')"
            v-if="recipe.image !== null"
            @load="onImgLoad"
          />
        </div>
      </div>

      <div class="row">
        <div class="col-12" style="text-align: center">
          <h3>{{ recipe.name }}</h3>
        </div>
      </div>

      <div class="row text-center">
        <div class="col col-md-12">
          <recipe-rating :recipe="recipe" />
          <last-cooked :recipe="recipe" class="mt-2" />
        </div>
      </div>

      <div class="my-auto">
        <div class="col-12" style="text-align: center">
          <i>{{ recipe.description }}</i>
        </div>
      </div>

      <div style="text-align: center">
        <keywords-component :recipe="recipe" />
      </div>

      <hr />
      <div class="row d-flex flex-row justify-content-around flex-nowrap">
        <div class="d-flex flex-row justify-content-center flex-wrap mx-5">
          <span class="text-primary mx-2">
            <b>{{ recipe.working_time }} {{ $t("min") }}</b>
          </span>
          <span>{{ $t("Preparation") }}</span>
        </div>

        <div class="d-flex flex-row justify-content-center flex-wrap mx-5">
          <span class="text-primary mx-2">
            <b>{{ recipe.waiting_time }} {{ $t("min") }}</b>
          </span>
          <span>{{ $t("Waiting") }}</span>
        </div>
      </div>
      <hr />

      <div class="row">
        <div class="col-12 order-2" v-if="recipe && ingredient_count > 0">
          <ingredients-card
            :recipe="recipe"
            :steps="recipe.steps"
            :ingredient_factor="ingredient_factor"
            :servings="servings"
            :header="true"
            id="ingredient_container"
            @checked-state-changed="updateIngredientCheckedState"
            @change-servings="servings = $event"
          />
        </div>

        <div
          class="col-12 order-1 col-sm-12 order-sm-1 col-md-6 order-md-2"
          v-if="recipe.nutrition !== null"
        >
          <div class="row" style="margin-top: 2vh; margin-bottom: 2vh">
            <div class="col-12">
              <nutrition-component
                :recipe="recipe"
                :ingredient_factor="ingredient_factor"
                id="nutrition_container"
              />
            </div>
          </div>
        </div>
      </div>

      <template v-if="!recipe.internal">
        <div v-if="recipe.file_path.includes('.pdf')">
          <pdf-viewer :recipe="recipe" />
        </div>
        <div
          v-if="
            recipe.file_path.includes('.png') ||
            recipe.file_path.includes('.jpg') ||
            recipe.file_path.includes('.jpeg') ||
            recipe.file_path.includes('.gif')
          "
        >
          <image-viewer :recipe="recipe" />
        </div>
      </template>

      <div
        v-for="(s, index) in recipe.steps"
        :key="s.id"
        style="margin-top: 1vh"
      >
        <step-component
          :recipe="recipe"
          :step="s"
          :ingredient_factor="ingredient_factor"
          :index="index"
          :start_time="start_time"
          @update-start-time="updateStartTime"
          @checked-state-changed="updateIngredientCheckedState"
        />
      </div>
    </div>

    <add-recipe-to-book :recipe="recipe" />

    <div
      class="row text-center d-print-none"
      style="margin-top: 3vh; margin-bottom: 3vh"
      v-if="share_uid !== 'None'"
    >
      <div class="col col-md-12">
        <a :href="resolveDjangoUrl('view_report_share_abuse', share_uid)">
          {{ $t("Report Abuse") }}
        </a>
      </div>
    </div>
  </div>
</template>

<script>
import Vue from "vue";
import { BootstrapVue } from "bootstrap-vue";
import moment from "moment";
import "bootstrap-vue/dist/bootstrap-vue.css";

import { apiLoadRecipe } from "@/utils/api";
import { ResolveUrlMixin, ToastMixin } from "@/utils/utils";

import RecipeContextMenu from "@/components/RecipeContextMenu";
import PdfViewer from "@/components/PdfViewer";
import ImageViewer from "@/components/ImageViewer";
import LoadingSpinner from "@/components/LoadingSpinner";
import AddRecipeToBook from "@/components/Modals/AddRecipeToBook";
import RecipeRating from "@/components/RecipeRating";
import LastCooked from "@/components/LastCooked";
import IngredientsCard from "@/components/IngredientsCard";
import StepComponent from "@/components/StepComponent";
import KeywordsComponent from "@/components/KeywordsComponent";
import NutritionComponent from "@/components/NutritionComponent";
import RecipeSwitcher from "@/components/Buttons/RecipeSwitcher";

Vue.prototype.moment = moment;

Vue.use(BootstrapVue);

export default {
  name: "RecipeView",
  mixins: [ResolveUrlMixin, ToastMixin],
  components: {
    LastCooked,
    RecipeRating,
    PdfViewer,
    ImageViewer,
    IngredientsCard,
    StepComponent,
    RecipeContextMenu,
    NutritionComponent,
    KeywordsComponent,
    LoadingSpinner,
    AddRecipeToBook,
    RecipeSwitcher,
  },
  computed: {
    ingredient_factor: function () {
      return this.servings / this.recipe.servings;
    },
    ingredient_count() {
      return this.recipe?.steps.map((x) => x.ingredients).flat().length;
    },
  },
  data() {
    return {
      loading: true,
      recipe: undefined,
      rootrecipe: undefined,
      servings: 1,
      servings_cache: {},
      start_time: "",
      share_uid: window.SHARE_UID,
      wake_lock: null,
      ingredient_height: "250",
    };
  },
  watch: {
    servings(newVal, oldVal) {
      this.servings_cache[this.recipe.id] = this.servings;
    },
  },
  mounted() {
    this.loadRecipe(window.RECIPE_ID);
    this.$i18n.locale = window.CUSTOM_LOCALE;
    this.requestWakeLock();
    window.addEventListener("resize", this.handleResize);
  },
  beforeUnmount() {
    this.destroyWakeLock();
  },

  methods: {
    requestWakeLock: async function () {
      if ("wakeLock" in navigator) {
        try {
          this.wake_lock = await navigator.wakeLock.request("screen");
          document.addEventListener("visibilitychange", this.visibilityChange);
        } catch (err) {
          console.log(err);
        }
      }
    },
    handleResize: function () {
      if (document.getElementById("nutrition_container") !== null) {
        this.ingredient_height =
          document.getElementById("ingredient_container").clientHeight -
          document.getElementById("nutrition_container").clientHeight;
      } else {
        this.ingredient_height = document.getElementById(
          "ingredient_container"
        ).clientHeight;
      }
    },
    destroyWakeLock: function () {
      if (this.wake_lock != null) {
        this.wake_lock.release().then(() => {
          this.wake_lock = null;
        });
      }

      document.removeEventListener("visibilitychange", this.visibilityChange);
    },
    visibilityChange: async function () {
      if (this.wake_lock != null && document.visibilityState === "visible") {
        await this.requestWakeLock();
      }
    },
    loadRecipe: function (recipe_id) {
      apiLoadRecipe(recipe_id).then((recipe) => {
        if (window.USER_SERVINGS !== 0) {
          recipe.servings = window.USER_SERVINGS;
        }

        let total_time = 0;
        for (let step of recipe.steps) {
          for (let ingredient of step.ingredients) {
            this.$set(ingredient, "checked", false);
          }

          step.time_offset = total_time;
          total_time += step.time;
        }

        // set start time only if there are any steps with timers (otherwise no timers are rendered)
        if (total_time > 0) {
          this.start_time = moment().format("yyyy-MM-DDTHH:mm");
        }
        if (recipe.image === null) this.printReady();

        this.recipe = this.rootrecipe = recipe;
        this.servings = this.servings_cache[this.rootrecipe.id] =
          recipe.servings;
        this.loading = false;

        setTimeout(() => {
          this.handleResize();
        }, 100);
      });
    },
    updateStartTime: function (e) {
      this.start_time = e;
    },
    updateIngredientCheckedState: function (e) {
      for (let step of this.recipe.steps) {
        for (let ingredient of step.ingredients) {
          if (ingredient.id === e.id) {
            this.$set(ingredient, "checked", !ingredient.checked);
          }
        }
      }
    },
    quickSwitch: function (e) {
      if (e === -1) {
        this.recipe = this.rootrecipe;
        this.servings = this.servings_cache[this.rootrecipe?.id ?? 1];
      } else {
        this.recipe = e;
        this.servings = this.servings_cache?.[e.id] ?? e.servings;
      }
    },
    printReady: function () {
      const template = document.createElement("template");
      template.id = "printReady";
      document.body.appendChild(template);
    },
    onImgLoad: function () {
      this.printReady();
    },
  },
};
</script>

<style>
#app > div > div {
  break-inside: avoid;
}

.recipe-header-image {
  object-fit: cover;
  width: 100%;
  top: 0;
  position: absolute;
  height: 100%;
  left: 0;
  display: block;
}

@media (min-width: 992px) {
  .recipe-header-image {
    border-radius: 0.25rem;
  }
}
</style>
