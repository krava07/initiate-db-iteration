<template>
    <div>
        <ul class="ruleset-list">
            <div class="placeholder-text" v-if="!value.rulesets.length">No query specified. All records will be returned as a result</div>
            <template v-for="(ruleset, index) in value.rulesets">
                <li :key="index" :class="['ruleset-item', { invalid: v[index].$error }]">
                    <div class="header display-flex">
                    <or-textbox
                        class="inline-input flex-1"
                        v-model="ruleset.title"
                        :disabled="readonly"
                    ></or-textbox>
                    <div class="display-flex align-items-center m-b-6">
                        <or-icon-button title="Duplicate" type="secondary" class="grey medium" icon="content_copy" @click="onDuplicate(ruleset)" :disabled="readonly"></or-icon-button>
                        <or-icon-button title="Toggle mode" type="secondary" class="grey" :disabled="ruleset.dirty || readonly" @click="toggleRulesetMode(ruleset)">
                            <div slot="default" class="svg-icon-position">
                                <svg width="24px" height="14px">
                                    <use xlink:href="#jsIcon"/>
                                </svg>
                            </div>
                        </or-icon-button>
                        <or-icon-button title="Delete" type="secondary" class="grey medium" icon="delete_forever" @click="onRemoveRuleset(ruleset)" :disabled="isDeleteDisabled"></or-icon-button>
                    </div>
                    </div>
                    <template v-if="ruleset.mode === 'select'">
                        <ul class="ruleset">
                            <rule
                                v-for="(rule, ruleIndex) in ruleset.rules"
                                :key="rule.id"
                                :rule="rule"
                                :steps="steps"
                                :step-id="stepId"
                                :operations="operations"
                                :readonly="readonly"
                                :fields="fields"
                                :table="table"
                                :ruleset="ruleset"
                                :v="v[index].rules.$each[ruleIndex]"
                                @ruleInput="(value) => onRuleInput(rule, value)"
                                @toggleRuleMode="toggleRuleMode(ruleset, rule)"
                                @removeRule="onRemoveRule(ruleset, rule)"
                            ></rule>
                        </ul>
                        <a href="#" @click.prevent="() => readonly || addRule(ruleset)" class="link-button" v-if="!readonly"><or-icon icon="add"></or-icon>add new rule</a>
                        <!-- <div v-if="v[index].$error && !v[index].hasIndexed" class="error">At least one indexed field should be included into the query.</div> -->
                        <div v-if="v[index].$error && !v[index].hasNoRepeatitions" class="error">The query cannot contain rules with equal fields and operations.</div>
                        <div v-if="v[index].$error && v[index].hasIndexed && v[index].hasNoRepeatitions" class="error">The query cannot contain rules with empty operations.</div>
                    </template>
                    <or-code
                        v-else
                        class="code-ruleset"
                        v-model="ruleset.code"
                        @input="(value) => onRulesetInput(ruleset, value)"
                        :readonly="readonly"
                        :steps="steps"
                        :step-id="stepId"
                        label=""
                    ></or-code>
                </li>
                <div :key="index" v-if="index !== value.rulesets.length - 1" class="separator">
                    <div>
                        <span>or</span>
                    </div>
                </div>
            </template>
        </ul>
        <a href="#" @click.prevent="() => readonly || addRuleset()" class="link-button" v-if="!readonly"><or-icon icon="add"></or-icon>add new filter</a>
    </div>
</template>

<script>
import rule from './rule.vue';
import uuid from 'uuid';
import {validators} from '_validators';

export default {
    // name: 'condition-builder',
    components: {
        rule
    },
    props: {
        value: {
            type: Object,
            default () {
                return {};
            }
        },
        fields: {
            type: Object,
            default () {
                return {};
            }
        },
        table: {
            type: Object,
            default: () => null
        },
        readonly: {
            type: Boolean,
            default: false
        },
        v: {
            type: Object,
            default: () => ({})
        },
        query: String,
        // query: {
        //   type: Object,
        //   default: () => ({})
        // },
        steps: {},
        stepId: {},
        useDataPrefix: {
            type: Boolean,
            default: false
        },
        allowRemoveLast: {
            type: Boolean,
            default: false
        }
    },
    data() {
        return {
            operations: [{
                label: 'equal',
                value: '='
            }, {
                label: '<=',
                value: '<='
            }, {
                label: '>=',
                value: '>='
            }, {
                label: '<',
                value: '<'
            }, {
                label: '>',
                value: '>'
            }, {
                label: 'not equal',
                value: '!='
            }]
            // {
            //   label: 'contains',
            //   value: 'contains',
            //   complex: true,
            //   getRule: expression => `$regex: \`\.*\${${expression}\}\.*\`\, $options: 'i'`
            // },
            // {
            //   label: 'not contain',
            //   value: 'not contain',
            //   complex: true,
            //   getRule: expression => `$regex: \`^((?!\${${expression}})\.)*$\`\, $options: 'i'`
            // },
            // {
            //   label: 'begins with',
            //   value: 'begins with',
            //   complex: true,
            //   getRule: expression => `$regex: \`^\${${expression}\}\`\, $options: 'i'`
            // }]
        };
    },
    computed: {
        isDeleteDisabled () {
            return (this.value.rulesets.length < 2 && !this.allowRemoveLast) || this.readonly;
        }
    },
    watch: {
        value: {
            handler(value) {
                const orExpressionsList = value.rulesets.map(this.rulesetToCode).filter(rs => !!rs);

                let query = '';
                if (orExpressionsList.length > 1) {
                    query = '(' + orExpressionsList.join(') OR (') + ')'
                } else {
                    query = orExpressionsList[0] || '';
                }
                // query = JSON.stringify(query);
                const fullQuery = '{ query_string: { query:' + '`' + query + '`' + '} }';
                this.$emit('update:query', fullQuery);
            },
            deep: true
        }
    },
    methods: {
        getEmptyRule() {
            return {
                id: uuid.v4(),
                field: '',
                operation: '=',
                expression: '``',
                mode: 'select',
                code: '',
                dirty: false
            };
        },
        addRule(ruleset) {
            ruleset.rules = [...ruleset.rules, this.getEmptyRule()];
        },
        addRuleset(clone) {
            this.value.rulesets = [...this.value.rulesets, clone || {
                title: 'New filter',
                rules: [this.getEmptyRule()],
                mode: 'select',
                code: '',
                dirty: false
            }];
        },

        onDuplicate(ruleset) {
            this.addRuleset(_.cloneDeep(ruleset));
        },
        toggleRulesetMode(ruleset) {
            ruleset.mode = ruleset.mode === 'code' ? 'select' : 'code';

            if (ruleset.mode === 'code') {
                ruleset.code = this.rulesetToCode(ruleset, 'select');
            }
        },
        toggleRuleMode(ruleset, rule) {
            rule.mode = rule.mode === 'code' ? 'select' : 'code';

            if (rule.mode === 'code') {
                rule.code = this.ruleToCode([rule], rule.field, 'select');
            }
        },
        onRuleInput(rule, value) {
            if (Object.keys(this.fields).length) {
                rule.dirty = this.ruleToCode([rule], rule.field, 'select') !== value;
            }
        },
        onRulesetInput(ruleset, value) {
            if (Object.keys(this.fields).length) {
                ruleset.dirty = this.rulesetToCode(ruleset, 'select') !== value;
            }
        },
        changeRule(fieldName, operator, value) {
            fieldName = fieldName.slice(1, -1);
            value = value.slice(1, -1);
            const notEquals = operator === '!=';
            const type = _.get(this.fields, `${fieldName}.type`);
            if (['!=', '='].includes(operator)) operator = '';
            if (type === 'keyword') {
                value = `"${value}"`;
                // value = JSON.stringify(value); 
            }
            let rule = `${fieldName} : ${operator}${value}`;
            if (notEquals) {
                rule = `!(${rule})`;
            }
            return rule;
        },
        ruleToCode(rules, field, source) {
            if ((source || rules[0].mode) === 'code') {
                return rules[0].code;
            }
            if (field && validators.jsExpressionNonEmptyString(field)) {
                // const operations = _.map(rules, this.oldChangeRule).filter(operation => !!operation).join(', ');
                const operations = _.map(rules, rule => this.changeRule(rule.field, rule.operation, rule.expression)).join(' AND ');

                if (!operations) {
                    return 'invalid js';
                }
                // return `[\`${this.useDataPrefix ? 'Data.' : ''}$\{${field}\}\`]: { ${operations} }`;
                // return '(' + operations + ')';
                return operations;
            }

            return '';
        },
        rulesetToCode(ruleset, source) {
            const {
                mode,
                code
            } = ruleset;

            if ((source || mode) === 'code') {
                return code;
            }

            const codeModeRules = _.filter(ruleset.rules, ({
                mode
            }) => mode === 'code');
            const selectModeRules = _.filter(ruleset.rules, ({
                mode
            }) => mode === 'select');
            const groupedRules = _.groupBy(selectModeRules, ({
                field
            }) => field);

            const rulesFields = [
                ..._.map(groupedRules, (rules, field) => this.ruleToCode(rules, field)),
                ..._.map(codeModeRules, rule => this.ruleToCode([rule]))
            ].filter(rule => !!rule).join(' AND ');

            return rulesFields || '';
        },
        onRemoveRule(ruleset, rule) {
            if (ruleset.rules.length < 2) {
                return;
            }
            ruleset.rules = _.reject(ruleset.rules, currRule => currRule === rule);
        },
        onRemoveRuleset(ruleset) {
            if (this.value.rulesets.length < 2 && !this.allowRemoveLast) {
                return;
            }
            this.value.rulesets = _.reject(this.value.rulesets, currRuleset => currRuleset === ruleset);
        }
    }
}
</script>

<style lang="scss">
.external-component-wrap {
    .inline-input {
        margin-bottom: 0;

        .ui-textbox__input {
            border: none;
            background-color: transparent;
            font-size: 14px;
            font-weight: bold;
            color: #0F232E;
            padding: 10px 18px 10px 0;
            margin-bottom: 9px;

            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;

            &:focus,
            &:hover {
                background-color: transparent;
            }
        }
    }
    

    %inline-select {
        .ui-select__display {
            border: none;
            background-color: transparent;
        }
    }

    .hide-controls {
        position: relative;

        .header {
            height: 0;
            min-height: 0;

            .js-mode-btn {
                display: none;
            }

            .add-variable {
                position: absolute;
                right: 0;
                // top: 36px;
                z-index: 1;
            }

            .full-screen {
                position: absolute;
                right: 0;
                bottom: 0;
                z-index: 1;
            }
        }
    }

    %or-filter-wrapper {
        background-color: #F6F6F6;
        padding: 6px 16px 16px 16px;
        border-radius: 3px;
        border-left: 3px solid #7ED321;

        &.invalid {
            border-left: 3px solid #F95D5D;
        }
    }

    %or-filter-item {
        position: relative;
        padding: 10px;
        border-radius: 3px;
        background-color: #FFF;
        align-items: stretch;
    }

    .ruleset-list {
        .separator {
            height: 61px;
            display: flex;
            justify-content: space-around;
            align-items: center;

            &>div {
                width: 27px;
                height: 27px;
                transform: rotate(45deg);
                display: flex;
                background-color: #7ED321;
                justify-content: space-around;
                align-items: center;

                &>span {
                    font-size: 12px;
                    line-height: 14px;
                    transform: rotate(-45deg);
                    text-transform: uppercase;
                    color: #FFF;
                }
            }
        }

        .ruleset-item {
            @extend %or-filter-wrapper;

            .ruleset {
                .rule {
                    @extend %or-filter-item;
                    padding: 7px;

                    .popover-trigger {
                        width: 10px;

                        .ui-icon {
                            position: relative;
                            left: -3px;
                        }
                    }

                    .field {
                        width: 0;

                        .ui-select__display {
                            height: 38px;
                            border-radius: 3px 0 0 3px;
                            padding-right: 5px;

                            .ui-icon.ui-select__dropdown-button {
                                width: 0.5em;

                                svg {
                                    position: relative;
                                    left: -0.25em;
                                }
                            }
                        }

                        .ui-select__content {
                            width: 0;
                        }

                        &.is-disabled {
                            .ui-icon.ui-select__dropdown-button {
                                display: none;
                            }
                        }
                    }

                    .operation {
                        &.error {
                            .ui-select__display-value {
                                color: #f95d5d;
                            }
                        }

                        min-width: 80px;
                        flex: 0.5;

                        .ui-select__dropdown {
                            min-width: auto;

                            .ui-select-option__basic {
                                font-size: 12px;
                            }
                        }

                        @extend %inline-select;

                        .ui-select__content {
                            width: 100%;

                            .ui-select__display {
                                padding-left: 7px;
                                padding-right: 7px;

                                .ui-select__display-value {
                                    text-align: center;
                                    font-size: 12px;

                                }

                                .ui-icon.ui-select__dropdown-button {
                                    width: 0.5em;

                                    svg {
                                        position: relative;
                                        left: -0.25em;
                                    }
                                }
                            }
                        }

                        &.is-disabled {
                            .ui-icon.ui-select__dropdown-button {
                                display: none;
                            }
                        }
                    }

                    .expression {
                        .or-editable-wrapper {
                            border-radius: 0 3px 3px 0;
                        }

                        width: 0;

                        .editable {
                            text-overflow: ellipsis;
                            padding-right: 30px;
                        }

                        &.readonly {
                            .editable {
                                padding-right: 10px;
                            }
                        }
                    }

                    &+.rule {
                        margin-top: 9px;
                    }
                }
            }
        }
    }
}
</style>
