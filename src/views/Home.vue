<template>
  <v-container class="mt-6">
    <v-row>
      <v-col cols="12" sm="12" md="6" lg="5" xl="4">
        <v-card
          flat
          outlined
          :class="{ 'sticky-preview': isStickyPreview }"
          id="preview-image-wrap"
        >
          <div class="d-flex">
            <v-card
              flat
              :width="previewWrapperWidth"
              :height="previewWrapperWidth"
              v-if="previewWrapperWidth !== 0"
            >
              <v-img
                class="preview-image"
                src="@/assets/blank-template.jpg"
                :width="previewWrapperWidth"
                :height="previewWrapperWidth"
                :aspect-ratio="1 / 1"
              />
              <VueDraggableResizable
                class="preview-image"
                w="auto"
                h="auto"
                :x="previewPosition.number.x"
                :y="previewPosition.number.y"
                :resizable="false"
                @dragging="dragNumber"
              >
                <div :style="previewStyle('number')" id="draggable-number">
                  {{ playerInfo.number }}
                </div>
              </VueDraggableResizable>
              <VueDraggableResizable
                class="preview-image"
                w="auto"
                h="auto"
                :x="previewPosition.name.x"
                :y="previewPosition.name.y"
                :resizable="false"
                @dragging="dragName"
              >
                <div :style="previewStyle('name')" id="draggable-name">
                  {{ playerInfo.name }}
                </div>
              </VueDraggableResizable>
              <v-img
                :width="previewWrapperWidth"
                :height="previewWrapperWidth"
                style="z-index: -10"
              />
            </v-card>
          </div>
        </v-card>
      </v-col>
      <v-col cols="12" sm="12" md="6" lg="7" xl="8">
        <v-card flat outlined>
          <v-card flat outlined class="setting-option ma-4 pa-4">
            <v-row class="player-info font-weight-bold text-center">
              <v-col cols="6">
                Player number
              </v-col>
              <v-col cols="6">
                Display name
              </v-col>
            </v-row>
            <v-row class="player-info">
              <v-col cols="6">
                <v-text-field
                  outlined
                  type="number"
                  v-model.number="playerInfo.number"
                  :hide-details="true"
                />
              </v-col>
              <v-col cols="6">
                <v-text-field
                  outlined
                  type="text"
                  v-model.number="playerInfo.name"
                  :hide-details="true"
                />
              </v-col>
            </v-row>
          </v-card>
          <v-card
            flat
            outlined
            class="setting-option ma-4 pa-4"
            v-if="currentFontEdit"
          >
            <div class="font-weight-bold">
              Editing player {{ currentFontEditLabel }}
            </div>
            <div class="d-flex toggle-edit">
              <v-btn
                elevation="1"
                :disabled="currentFontEditLabel === 'name'"
                @click="currentFontEditLabel = 'name'"
              >
                Player name
              </v-btn>
              <v-btn
                elevation="1"
                :disabled="currentFontEditLabel === 'number'"
                @click="currentFontEditLabel = 'number'"
              >
                Player number
              </v-btn>
            </div>
            <v-row class="d-flex" justify="space-between">
              <v-col cols="3">Color:</v-col>
              <v-col cols="8" class="preview-color pt-1">
                <input
                  type="color"
                  v-debounce:30="setFontColor"
                  id="color-picker"
                  value="#ffffff"
                />
              </v-col>
            </v-row>
            <v-row class="font-option">
              <v-col cols="12" sm="6" md="6" lg="4" xl="3">
                <v-autocomplete
                  outlined
                  :items="fontOptions"
                  label="Choose font"
                  v-model="currentFontEdit.font"
                >
                  <template v-slot:append-item>
                    <v-divider />
                    <v-file-input
                      :rules="imageRule"
                      class="upload-font"
                      outlined
                      accept=".ttf"
                      placeholder="Or upload a .ttf font"
                      prepend-icon="mdi-format-font"
                      v-model="newFont"
                      @change="uploadFont"
                      hide-details="auto"
                      :error-messages="uploadFontErrorMessage"
                    />
                  </template>
                </v-autocomplete>
              </v-col>
              <v-col cols="12" sm="6" md="6" lg="4" xl="3">
                <v-text-field
                  outlined
                  label="Font size"
                  placeholder="Font size"
                  type="number"
                  v-model="currentFontEdit.fontSize"
                  hide-details="auto"
                />
              </v-col>
              <v-col cols="12" sm="6" md="6" lg="4" xl="3">
                <v-text-field
                  outlined
                  label="Text height"
                  placeholder="Text height"
                  type="number"
                  v-model="currentFontEdit.height"
                  hide-details="auto"
                />
              </v-col>
              <v-col cols="12" sm="6" md="6" lg="4" xl="3">
                <v-text-field
                  outlined
                  label="Width offset"
                  placeholder="Width offset"
                  type="number"
                  v-model="currentFontEdit.widthOffset"
                  hide-details="auto"
                />
              </v-col>
            </v-row>
          </v-card>
        </v-card>
      </v-col>
    </v-row>
  </v-container>
</template>

<script lang="ts">
/* eslint-disable @typescript-eslint/no-explicit-any */
/* eslint-disable @typescript-eslint/no-var-requires */
/* eslint-disable @typescript-eslint/explicit-module-boundary-types */
import Vue from "vue";
import { Watch, Component } from "vue-property-decorator";
import { Debounce } from "vue-debounce-decorator";
import { Setting, Player } from "@/models/setting";
const VueDraggableResizable = require("vue-draggable-resizable");
import vueDebounce from "vue-debounce";

Vue.use(vueDebounce, {
  lock: true,
  defaultTime: "400ms",
  listenTo: "input"
});

@Component({
  components: {
    VueDraggableResizable
  }
})
export default class MakeYourShirt extends Vue {
  private uploadBaseImageErrorMessage = "";
  private uploadFontErrorMessage = "";

  private setting = new Setting();
  private isStickyPreview = false;
  private previewWrapperWidth = 0;
  private imageRule = [
    (value: File) =>
      !value || value.size < 1000000 || "Image size needs to be less than 1 MB!"
  ];
  private playerInfo = new Player();

  private newImage = {
    template: null,
    keeperTemplate: null
  };
  private newFont: File | null = null;

  private currentFontEditLabel: "name" | "number" = "name";
  private fontLabels: ["number", "name"] = ["number", "name"];

  private previewPosition = {
    number: { x: 0, y: 0 },
    name: { x: 0, y: 0 }
  };
  private previewBaseImage = require("@/assets/blank-template.jpg");

  async created() {
    this.setStickyPreview();
    await this.loadFonts();
    await this.setPreviewWrapper();
    await this.setPreviewPosition();
    window.addEventListener("resize", this.onContainerWidthChanged);
  }

  destroyed(): void {
    window.removeEventListener("resize", this.onContainerWidthChanged);
  }

  async loadFonts() {
    await Promise.all(
      this.fontOptions.map(async font => {
        const fontStyle = new FontFace(
          font,
          `url(${require("@/assets/fonts/" + font + ".ttf")})`
        );
        document.fonts.add(await fontStyle.load());
      })
    );
  }

  setFontColor(value: string, event: any): void {
    const result = /^#?([a-f\d]{2})([a-f\d]{2})([a-f\d]{2})$/i.exec(value);
    if (result) {
      this.currentFontEdit.color = [
        parseInt(result[1], 16),
        parseInt(result[2], 16),
        parseInt(result[3], 16)
      ];
    }
  }

  previewStyle(fontEditLabel: "number" | "name"): CSSStyleDeclaration {
    return {
      fontSize:
        (this.setting[fontEditLabel].fontSize / 960) *
          this.previewWrapperWidth +
        "px",
      color: `rgb(${this.setting[fontEditLabel].color.join(", ")})`,
      fontFamily: `"${this.fontFamily[fontEditLabel]}"`,
      webkitTextStroke: "#622 0.5px"
    } as CSSStyleDeclaration;
  }

  setStickyPreview(): void {
    const breakpoint = this.$vuetify.breakpoint;
    this.isStickyPreview = !(breakpoint.xs || breakpoint.sm);
  }

  async onContainerWidthChanged(): Promise<void> {
    await new Promise(resolve => setTimeout(resolve, 500));
    await this.setPreviewWrapper();
    this.setPreviewPosition();
  }

  async setPreviewWrapper(): Promise<void> {
    await Vue.nextTick();
    this.previewWrapperWidth =
      document.getElementById("preview-image-wrap")?.clientWidth || 0;
  }

  dragNumber(left: number, top: number): void {
    this.onDrag(left, top, "number");
  }
  dragName(left: number, top: number): void {
    this.onDrag(left, top, "name");
  }
  @Debounce(50)
  onDrag(
    left: number,
    top: number,
    currentFontEditLabel: "number" | "name"
  ): void {
    const currentFontEdit = this.setting[currentFontEditLabel];
    const previewDiv = this.previewEditable[currentFontEditLabel];
    if (previewDiv) {
      currentFontEdit.height = Math.round(
        ((top + previewDiv.clientHeight * 0.42) / this.previewWrapperWidth) *
          1920
      );
      currentFontEdit.widthOffset = Math.round(
        ((left + previewDiv.clientWidth / 2 - this.previewWrapperWidth / 2) /
          this.previewWrapperWidth) *
          960
      );
    }
  }

  get previewEditable(): any {
    return {
      number: document.getElementById("draggable-number"),
      name: document.getElementById("draggable-name")
    };
  }

  get baseImages() {
    return this.$store.getters["mockup/editMockup/getBaseImages"];
  }

  get currentFontEdit() {
    return this.setting[this.currentFontEditLabel];
  }
  @Watch("currentFontEdit", { deep: true })
  onCurrentFontEditChanged() {
    const colorPicker: any = document.getElementById("color-picker");
    const rgb: number[] = this.currentFontEdit.color;
    if (colorPicker)
      colorPicker.value =
        "#" +
        ((1 << 24) + (rgb[0] << 16) + (rgb[1] << 8) + rgb[2])
          .toString(16)
          .slice(1);
    this.setPreviewPosition();
  }

  get fontFamily() {
    return {
      number: this.setting.number.font,
      name: this.setting.name.font
    };
  }
  @Watch("fontFamily", { deep: true })
  async setPreviewPosition(): Promise<void> {
    await Vue.nextTick();
    this.fontLabels.forEach(label => {
      const previewDiv = this.previewEditable[label];
      if (previewDiv) {
        this.previewPosition[label].x = Math.round(
          (this.setting[label].widthOffset / 960) * this.previewWrapperWidth +
            this.previewWrapperWidth / 2 -
            previewDiv.clientWidth / 2
        );
        this.previewPosition[label].y = Math.round(
          (this.setting[label].height / 1920) * this.previewWrapperWidth -
            previewDiv.clientHeight * 0.42
        );
      }
    });
  }
  @Watch("playerInfo", { deep: true })
  async setPlayerInfoPosition(): Promise<void> {
    await this.setPreviewPosition();
  }

  async uploadFont(event: File | null) {
    if (event) {
      if (!this.fontOptions.some((x: any) => x === event.name)) {
        const fontFamily = event.name
          .split(".")
          .slice(0, -1)
          .join(".");
        const newFontStyle = new FontFace(
          fontFamily,
          await event.arrayBuffer()
        );
        console.log(await newFontStyle.load());
        document.fonts.add(await newFontStyle.load());
        this.currentFontEdit.font = fontFamily;
        this.uploadFontErrorMessage = "";
        return;
      }
      this.uploadFontErrorMessage = `Font file ${event.name} already existed. Please rename the file before uploading.`;
      this.newFont = null;
    }
    this.currentFontEdit.font = "AmaticSC-Regular";
  }

  get fontOptions(): string[] {
    const assetsFont = [
      "AmaticSC-Regular",
      "JosefinSans-Regular",
      "Raleway-Regular",
      "SEASRN__"
    ];
    if (this.newFont) {
      return [
        ...assetsFont,
        this.newFont.name.split(".").slice(0, -1).join(".")
      ]
    }
    return assetsFont
  }
}
</script>

<style lang="scss">
.sticky-preview {
  position: sticky;
  // top: 5rem;

  .preview-image {
    position: absolute;
    user-select: none;
  }
}

.v-input--radio-group__input {
  justify-content: space-evenly;
}

.setting-option {
  .image-scroll {
    display: flex;
    overflow-x: auto;

    .base-image {
      margin: 1rem;
      border: 0.1rem solid #999;
      border-radius: 1rem;
      height: 12.5rem;
      width: 12.5rem;

      &-selected {
        border: 0.2rem solid #88f;
      }
    }

    .choose-logo {
      height: 7rem;
      width: 7rem;
    }
  }

  .toggle-edit {
    margin: 1rem auto 2rem;

    button {
      width: 50%;
      border-radius: 0;
    }
  }

  .preview-color {
    width: 100%;

    #color-picker {
      width: 100%;
      height: 2.6rem;
    }
  }

  .color-option {
    padding-top: 1rem;

    > * {
      padding: 0 12px;
    }
  }

  .upload-font {
    margin: 0.7rem 0.5rem auto !important;
  }
}
</style>