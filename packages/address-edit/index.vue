<template>
  <div :class="b()">
    <cell-group>
      <field
        v-model="data.name"
        maxlength="15"
        :label="$t('name')"
        :placeholder="$t('namePlaceholder')"
        :error="errorInfo.name"
        @focus="onFocus('name')"
      />
      <field
        v-model="data.tel"
        type="tel"
        :label="$t('tel')"
        :placeholder="$t('telPlaceholder')"
        :error="errorInfo.tel"
        @focus="onFocus('tel')"
      />
      <field
        readonly
        :label="$t('area')"
        :placeholder="$t('areaPlaceholder')"
        :value="areaText"
        @click="showArea = true"
      />
      <address-edit-detail
        :focused="detailFocused"
        :value="data.addressDetail"
        :error="errorInfo.addressDetail"
        :search-result="searchResult"
        :show-search-result="showSearchResult"
        @focus="onFocus('addressDetail')"
        @blur="detailFocused = false"
        @input="onChangeDetail"
        @select-search="$emit('select-search', $event)"
      />
      <field
        v-if="showPostal"
        v-show="!hideBottomFields"
        v-model="data.postalCode"
        type="tel"
        maxlength="6"
        class="van-hairline--top"
        :label="$t('postal')"
        :placeholder="$t('postal')"
        :error="errorInfo.postalCode"
        @focus="onFocus('postalCode')"
      />
      <slot />
      <switch-cell
        v-if="showSetDefault"
        v-show="!hideBottomFields"
        v-model="data.isDefault"
        :title="$t('defaultAddress')"
      />
    </cell-group>

    <div v-show="!hideBottomFields" :class="b('buttons')">
      <van-button block :loading="isSaving" @click="onSave" type="danger">
        {{ saveButtonText || $t('save') }}
      </van-button>
      <van-button block :loading="isDeleting" @click="onDelete" v-if="showDelete">
        {{ deleteButtonText || $t('delete') }}
      </van-button>
    </div>

    <popup v-model="showArea" position="bottom" :lazy-render="false" :get-container="getAreaContainer">
      <van-area
        ref="area"
        :loading="!areaListLoaded"
        :value="data.areaCode"
        :area-list="areaList"
        @confirm="onAreaConfirm"
        @cancel="showArea = false"
      />
    </popup>
  </div>
</template>

<script>
/* eslint-disable camelcase */
import create from '../utils/create';
import { isObj } from '../utils';
import Field from '../field';
import VanButton from '../button';
import Popup from '../popup';
import Toast from '../toast';
import Dialog from '../dialog';
import VanArea from '../area';
import AddressEditDetail from './Detail';
import SwitchCell from '../switch-cell';
import validateMobile from '../utils/validate/mobile';

const defaultData = {
  name: '',
  tel: '',
  province: '',
  city: '',
  county: '',
  areaCode: '',
  postalCode: '',
  addressDetail: '',
  isDefault: false
};

export default create({
  name: 'address-edit',

  components: {
    Field,
    Popup,
    VanArea,
    VanButton,
    SwitchCell,
    AddressEditDetail
  },

  props: {
    areaList: Object,
    isSaving: Boolean,
    isDeleting: Boolean,
    showDelete: Boolean,
    showPostal: Boolean,
    showSetDefault: Boolean,
    showSearchResult: Boolean,
    saveButtonText: String,
    deleteButtonText: String,
    addressInfo: {
      type: Object,
      default: () => ({ ...defaultData })
    },
    searchResult: {
      type: Array,
      default: () => []
    },
    telValidator: {
      type: Function,
      default: validateMobile
    }
  },

  data() {
    return {
      data: {},
      showArea: false,
      detailFocused: false,
      errorInfo: {
        tel: false,
        name: false,
        postalCode: false,
        addressDetail: false
      }
    };
  },

  computed: {
    // hide bottom field when use search && detail get focused
    hideBottomFields() {
      return this.searchResult.length && this.detailFocused;
    },

    areaListLoaded() {
      return isObj(this.areaList) && Object.keys(this.areaList).length;
    },

    areaText() {
      const { province, city, county, areaCode } = this.data;
      return province && city && county && areaCode
        ? `${province} ${city} ${county}`
        : '';
    }
  },

  watch: {
    addressInfo: {
      handler(val) {
        this.data = {
          ...defaultData,
          ...val
        };

        this.setAreaCode(val.areaCode);
      },
      deep: true,
      immediate: true
    },

    areaList() {
      this.setAreaCode(this.data.areaCode);
    }
  },

  methods: {
    onFocus(key) {
      this.errorInfo[key] = false;
      this.detailFocused = key === 'addressDetail';
      this.$emit('focus', key);
    },

    onChangeDetail(val) {
      this.data.addressDetail = val;
      this.$emit('change-detail', val);
    },

    onAreaConfirm(values) {
      this.showArea = false;
      this.data.areaCode = values[2].code;
      this.assignAreaValues(values);
      this.$emit('change-area', values);
    },

    assignAreaValues(values) {
      Object.assign(this.data, {
        province: values[0] ? values[0].name : '',
        city: values[1] ? values[1].name : '',
        county: values[2] ? values[2].name : ''
      });
    },

    onSave() {
      const items = ['name', 'tel', 'areaCode', 'addressDetail'];

      if (this.showPostal) {
        items.push('postalCode');
      }

      const isValid = items.every(item => {
        const msg = this.getErrorMessage(item);
        if (msg) {
          this.errorInfo[item] = true;
          Toast(msg);
        }
        return !msg;
      });

      if (isValid && !this.isSaving) {
        this.$emit('save', this.data);
      }
    },

    getErrorMessage(key) {
      const value = String(this.data[key]).trim();
      const { $t } = this;

      switch (key) {
        case 'name':
          return value ? '' : $t('nameEmpty');
        case 'tel':
          return this.telValidator(value) ? '' : $t('telInvalid');
        case 'areaCode':
          return value ? '' : $t('areaEmpty');
        case 'addressDetail':
          return value ? '' : $t('addressEmpty');
        case 'postalCode':
          return value && !/^\d{6}$/.test(value) ? $t('postalEmpty') : '';
      }
    },

    onDelete() {
      Dialog.confirm({
        title: this.$t('confirmDelete')
      })
        .then(() => {
          this.$emit('delete', this.data);
        })
        .catch(() => {
          this.$emit('cancel-delete', this.data);
        });
    },

    // get values of area component
    getArea() {
      return this.$refs.area ? this.$refs.area.getValues() : [];
    },

    // set area code to area component
    setAreaCode(code) {
      this.data.areaCode = code || '';
      this.$nextTick(() => {
        const { area } = this.$refs;
        if (area) {
          this.assignAreaValues(area.getValues());
        }
      });
    },

    getAreaContainer() {
      return document.body;
    }
  }
});
</script>
