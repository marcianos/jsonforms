<template>
  <dispatch-renderer
    v-bind:schema="jsonforms.core.schema"
    v-bind:uischema="jsonforms.core.uischema"
    v-bind:path="''"
  />
</template>

<script lang="ts">
import { PropType } from 'vue';
import { defineComponent } from '../../config';
import {
  coreReducer,
  Actions,
  Generate,
  configReducer,
  JsonSchema,
  UISchemaElement,
  ValidationMode,
  JsonFormsCore,
  JsonFormsUISchemaRegistryEntry,
  JsonFormsRendererRegistryEntry,
  JsonFormsCellRendererRegistryEntry,
  CoreActions,
  i18nReducer,
  JsonFormsI18nState
} from '@jsonforms/core';
import { JsonFormsChangeEvent, MaybeReadonly } from '../types';
import DispatchRenderer from './DispatchRenderer.vue';

import Ajv, { ErrorObject } from 'ajv';

const isObject = (elem: any): elem is Object => {
  return elem && typeof elem === 'object';
};

export default defineComponent({
  name: 'json-forms',
  components: {
    DispatchRenderer
  },
  emits: ['change'],
  props: {
    data: {
      required: true,
      type: [String, Number, Boolean, Array, Object] as PropType<any>
    },
    schema: {
      required: false,
      type: [Object, Boolean] as PropType<JsonSchema>,
      default: undefined
    },
    uischema: {
      required: false,
      type: Object as PropType<UISchemaElement>,
      default: undefined
    },
    renderers: {
      required: true,
      type: Array as PropType<MaybeReadonly<JsonFormsRendererRegistryEntry[]>>
    },
    cells: {
      required: false,
      type: Array as PropType<MaybeReadonly<JsonFormsCellRendererRegistryEntry[]>>,
      default: () => []
    },
    config: {
      required: false,
      type: Object as PropType<any>,
      default: undefined
    },
    readonly: {
      required: false,
      type: Boolean,
      default: false
    },
    uischemas: {
      required: false,
      type: Array as PropType<MaybeReadonly<JsonFormsUISchemaRegistryEntry[]>>,
      default: () => []
    },
    validationMode: {
      required: false,
      type: String as PropType<ValidationMode>,
      default: 'ValidateAndShow'
    },
    ajv: {
      required: false,
      type: Object as PropType<Ajv>,
      default: undefined
    },
    i18n: {
      required: false,
      type: Object as PropType<JsonFormsI18nState>,
      default: undefined
    },
    additionalErrors: {
      required: false,
      type: Array as PropType<ErrorObject[]>,
      default: () => []
    },
  },
  data() {
    const generatorData = isObject(this.data) ? this.data : {};
    const schemaToUse = this.schema ?? Generate.jsonSchema(generatorData);
    const uischemaToUse = this.uischema ?? Generate.uiSchema(schemaToUse);
    const initCore = (): JsonFormsCore => {
      const initialCore = {
        data: this.data,
        schema: schemaToUse,
        uischema: uischemaToUse
      };
      const core = coreReducer(
        initialCore,
        Actions.init(this.data, schemaToUse, uischemaToUse, {
          validationMode: this.validationMode,
          ajv: this.ajv,
          additionalErrors: this.additionalErrors
        })
      );
      return core;
    };
    return {
      schemaToUse,
      uischemaToUse,
      jsonforms: {
        core: initCore(),
        config: configReducer(undefined, Actions.setConfig(this.config)),
        i18n: i18nReducer(this.i18n, Actions.updateI18n(this.i18n?.locale, this.i18n?.translate, this.i18n?.translateError)),
        renderers: this.renderers,
        cells: this.cells,
        uischemas: this.uischemas,
        readonly: this.readonly
      }
    };
  },
  watch: {
    schema(newSchema) {
      const generatorData = isObject(this.data) ? this.data : {};
      this.schemaToUse = newSchema ?? Generate.jsonSchema(generatorData);
      if (!this.uischema) {
        this.uischemaToUse = Generate.uiSchema(this.schemaToUse);
      }
    },
    uischema(newUischema) {
      this.uischemaToUse = newUischema ?? Generate.uiSchema(this.schemaToUse);
    },
    renderers(newRenderers) {
      this.jsonforms.renderers = newRenderers;
    },
    cells(newCells) {
      this.jsonforms.cells = newCells;
    },
    uischemas(newUischemas) {
      this.jsonforms.uischemas = newUischemas;
    },
    config: {
      handler(newConfig) {
        this.jsonforms.config = configReducer(
          undefined,
          Actions.setConfig(newConfig)
        );
      },
      deep: true
    },
    readonly(newReadonly) {
      this.jsonforms.readonly = newReadonly;
    },
    coreDataToUpdate() {
      this.jsonforms.core = coreReducer(
        this.jsonforms.core as JsonFormsCore,
        Actions.updateCore(this.data, this.schemaToUse, this.uischemaToUse, {
          validationMode: this.validationMode,
          ajv: this.ajv,
          additionalErrors: this.additionalErrors
        })
      );
    },
    eventToEmit(newEvent) {
      this.$emit('change', newEvent);
    },
    i18n: {
      handler(newI18n) {
        this.jsonforms.i18n = i18nReducer(
          this.jsonforms.i18n,
          Actions.updateI18n(newI18n?.locale, newI18n?.translate, newI18n?.translateError)
        );
      },
      deep: true
    }
  },
  computed: {
    coreDataToUpdate(): any {
      return [
        this.data,
        this.schemaToUse,
        this.uischemaToUse,
        this.validationMode,
        this.ajv,
        this.additionalErrors
      ];
    },
    eventToEmit(): JsonFormsChangeEvent {
      return {
        data: this.jsonforms.core.data,
        errors: this.jsonforms.core.errors
      };
    }
  },
  mounted() {
    // emit an inital change so clients can react to error validation and default data insertion
    this.$emit('change', {
      data: this.jsonforms.core.data,
      errors: this.jsonforms.core.errors
    });
  },
  methods: {
    dispatch(action: CoreActions) {
      this.jsonforms.core = coreReducer(this.jsonforms.core as JsonFormsCore, action);
    }
  },
  provide() {
    return {
      jsonforms: this.jsonforms,
      dispatch: this.dispatch
    };
  }
});
</script>
