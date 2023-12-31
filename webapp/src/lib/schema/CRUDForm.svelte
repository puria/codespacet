<script lang="ts" context="module">
	import type { RelationDisplayFields } from '$lib/components/forms/relations.svelte';
	export type RelationsDisplayFields = Record<string, RelationDisplayFields>;
</script>

<script lang="ts">
	import Form, { createForm } from '$lib/components/forms/form.svelte';
	import { getCollectionSchema } from './getCollectionSchema';
	import { fieldsSchemaToZod } from './collectionSchemaToZod';
	import type { Collections } from '$lib/pocketbase-types';
	import FieldSchemaToInput from './fieldSchemaToInput.svelte';
	import { FieldType, type FieldSchema } from './types';
	import { pb } from '$lib/pocketbase';
	import { log } from 'debug';
	import { createEventDispatcher } from 'svelte';

	//

	export let collection: Collections;
	export let initialData: any = undefined;

	export let fieldsOrder: string[] = [];
	export let hiddenFields: string[] = [];
	export let excludedFields: string[] = [];
	export let relationsDisplayFields: RelationsDisplayFields = {};

	//

	const dispatch = createEventDispatcher<{ success: {} }>();

	const collectionSchema = getCollectionSchema(collection)!;
	const fieldsSchema = collectionSchema.schema
		.sort(sortFieldsSchema)
		.filter(filterFieldsSchema)
		.filter(excludeMultiselect);
	const zodSchema = fieldsSchemaToZod(fieldsSchema);

	const superform = createForm(
		zodSchema,
		async ({ form }) => {
			const formData = createFormData(form.data);
			if (initialData && initialData.id) {
				await pb.collection(collection).update(initialData.id, formData);
			} else {
				await pb.collection(collection).create(formData);
			}
			dispatch('success', {});
		},
		initialData
	);

	//

	function sortFieldsSchema(a: any, b: any) {
		const aIndex = fieldsOrder.indexOf(a.name);
		const bIndex = fieldsOrder.indexOf(b.name);
		if (aIndex === -1 && bIndex === -1) {
			return 0;
		}
		if (aIndex === -1) {
			return 1;
		}
		if (bIndex === -1) {
			return -1;
		}
		return aIndex - bIndex;
	}

	function filterFieldsSchema(schema: FieldSchema) {
		return !excludedFields.includes(schema.name);
	}

	function excludeMultiselect(schema: FieldSchema) {
		if (schema.type == FieldType.SELECT && (schema.options.maxSelect as number) > 1) {
			log('multiple select not supported yet');
			return false;
		}
		return true;
	}

	function createFormData(data: Record<string, unknown>) {
		const formData = new FormData();
		for (const [key, value] of Object.entries(data)) {
			if (value === null || value === undefined) {
				// Needed otherwise pb complains about "bad formatting", especially for null files
				continue;
			} else if (Array.isArray(value)) {
				// Special case for empty arrays, cause they can't be represented in formData
				if (value.length === 0) {
					formData.append(`${key}-`, initialData[key]);
				} else {
					for (const item of value) {
						formData.append(key, item);
					}
				}
			} else {
				formData.append(key, value as string | File);
			}
		}
		return formData;
	}

	//

	const defaultSubmitButtonText = Boolean(initialData) ? 'Edit record' : 'Create record';
</script>

<Form {superform} {defaultSubmitButtonText} on:success>
	{#each fieldsSchema as fieldSchema}
		{@const hidden = hiddenFields.includes(fieldSchema.name)}
		{@const relationDisplayFields = relationsDisplayFields[fieldSchema.name] || []}
		<FieldSchemaToInput {fieldSchema} {hidden} {relationDisplayFields} />
	{/each}
</Form>
