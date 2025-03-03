<template>
	<div @keydown.stop class="collection-parameter">
		<div class="collection-parameter-wrapper">
			<div v-if="getProperties.length === 0" class="no-items-exist">
				Currently no properties exist
			</div>

			<parameter-input-list :parameters="getProperties" :nodeValues="nodeValues" :path="path" :hideDelete="hideDelete" @valueChanged="valueChanged" />

			<div v-if="parameterOptions.length > 0 && !isReadOnly" class="param-options">
				<n8n-button
					v-if="parameter.options.length === 1"
					fullWidth
					@click="optionSelected(parameter.options[0].name)"
					:label="getPlaceholderText"
				/>
				<div v-else class="add-option">
					<n8n-select v-model="selectedOption" :placeholder="getPlaceholderText" size="small"  @change="optionSelected" filterable>
						<n8n-option
							v-for="item in parameterOptions"
							:key="item.name"
							:label="item.displayName"
							:value="item.name">
						</n8n-option>
					</n8n-select>
				</div>
			</div>

		</div>
	</div>
</template>

<script lang="ts">
import {
	IUpdateInformation,
} from '@/Interface';

import {
	INodeProperties,
	INodePropertyOptions,
} from 'n8n-workflow';

import { genericHelpers } from '@/components/mixins/genericHelpers';
import { nodeHelpers } from '@/components/mixins/nodeHelpers';

import { get } from 'lodash';

import mixins from 'vue-typed-mixins';

export default mixins(
	genericHelpers,
	nodeHelpers,
)
	.extend({
		name: 'CollectionParameter',
		props: [
			'hideDelete', // boolean
			'nodeValues', // NodeParameters
			'parameter', // INodeProperties
			'path', // string
			'values', // NodeParameters
		],
		data () {
			return {
				selectedOption: undefined,
			};
		},
		computed: {
			getPlaceholderText (): string {
				return this.parameter.placeholder ? this.parameter.placeholder : 'Choose Option To Add';
			},
			getProperties (): INodeProperties[] {
				const returnProperties = [];
				let tempProperties;
				for (const name of this.propertyNames) {
					tempProperties = this.getOptionProperties(name);
					if (tempProperties !== undefined) {
						returnProperties.push(...tempProperties);
					}
				}
				return returnProperties;
			},
			// Returns all the options which should be displayed
			filteredOptions (): Array<INodePropertyOptions | INodeProperties> {
				return (this.parameter.options as Array<INodePropertyOptions | INodeProperties>).filter((option) => {
					return this.displayNodeParameter(option as INodeProperties);
				});
			},
			// Returns all the options which did not get added already
			parameterOptions (): Array<INodePropertyOptions | INodeProperties> {
				return (this.filteredOptions as Array<INodePropertyOptions | INodeProperties>).filter((option) => {
					return !this.propertyNames.includes(option.name);
				});
			},
			propertyNames (): string[] {
				if (this.values) {
					return Object.keys(this.values);
				}
				return [];
			},
		},
		methods: {
			getArgument (argumentName: string): string | number | boolean | undefined {
				if (this.parameter.typeOptions === undefined) {
					return undefined;
				}

				if (this.parameter.typeOptions[argumentName] === undefined) {
					return undefined;
				}

				return this.parameter.typeOptions[argumentName];
			},
			getOptionProperties (optionName: string): INodeProperties[] {
				const properties: INodeProperties[] = [];
				for (const option of this.parameter.options) {
					if (option.name === optionName) {
						properties.push(option);
					}
				}

				return properties;
			},
			displayNodeParameter (parameter: INodeProperties) {
				if (parameter.displayOptions === undefined) {
					// If it is not defined no need to do a proper check
					return true;
				}
				return this.displayParameter(this.nodeValues, parameter, this.path);
			},
			optionSelected (optionName: string) {
				const options = this.getOptionProperties(optionName);
				if (options.length === 0) {
					return;
				}

				const option = options[0];
				const name = `${this.path}.${option.name}`;

				let parameterData;

				if (option.typeOptions !== undefined && option.typeOptions.multipleValues === true) {
					// Multiple values are allowed

					let newValue;
					if (option.type === 'fixedCollection') {
						// The "fixedCollection" entries are different as they save values
						// in an object and then underneath there is an array. So initialize
						// them differently.
						newValue = get(this.nodeValues, `${this.path}.${optionName}`, {});
					} else {
						// Everything else saves them directly as an array.
						newValue = get(this.nodeValues, `${this.path}.${optionName}`, []);
						newValue.push(JSON.parse(JSON.stringify(option.default)));
					}

					parameterData = {
						name,
						value: newValue,
					};
				} else {
					// Add a new option
					parameterData = {
						name,
						value: JSON.parse(JSON.stringify(option.default)),
					};
				}

				this.$emit('valueChanged', parameterData);
				this.selectedOption = undefined;
			},
			valueChanged (parameterData: IUpdateInformation) {
				this.$emit('valueChanged', parameterData);
			},
		},
		beforeCreate: function () { // tslint:disable-line
			// Because we have a circular dependency on ParameterInputList import it here
			// to not break Vue.
			this.$options!.components!.ParameterInputList = require('./ParameterInputList.vue').default;
		},
	});
</script>

<style lang="scss">

.collection-parameter {
	padding-left: 2em;

	.param-options {
		padding-top: 0.5em;
	}

	.no-items-exist {
		margin: 0.8em 0 0.4em 0;
	}
	.option {
		position: relative;
		padding: 0.25em 0 0.25em 1em;
	}
}

</style>
