<template>
	<div class="content">
		<div v-for="group of groups" :key="group.key" class="group">
			<h2>{{ group.name }}</h2>

			<div class="grid">
				<button
					v-for="inter of group.interfaces"
					:key="inter.id"
					class="interface"
					:class="{ active: chosenInterface === inter.id, gray: chosenInterface && chosenInterface !== inter.id }"
					@click="toggleInterface(inter.id)"
				>
					<div class="preview">
						<template v-if="inter.preview">
							<!-- eslint-disable-next-line vue/no-v-html -->
							<span v-if="isSVG(inter.preview)" v-html="inter.preview" />
							<img v-else :src="inter.preview" alt="" />
						</template>

						<span v-else class="fallback">
							<v-icon large :name="inter.icon" />
						</span>
					</div>
					<v-text-overflow :text="inter.name" class="name" />
				</button>

				<transition-expand>
					<field-configuration
						v-if="chosenInterface && !!group.interfaces.some((inter) => inter.id === chosenInterface)"
						:row="configRow"
						@save="$emit('save')"
						@toggleAdvanced="$emit('toggleAdvanced')"
					/>
				</transition-expand>
			</div>
		</div>
	</div>
</template>

<script lang="ts">
import { defineComponent, PropType, computed, toRefs, watch } from 'vue';
import { Collection } from '@directus/shared/types';
import { useI18n } from 'vue-i18n';
import { getInterfaces } from '@/interfaces';
import { orderBy } from 'lodash';
import { useFieldDetailStore, syncFieldDetailStoreProperty } from '../store/';
import { syncRefProperty } from '@/utils/sync-ref-property';
import FieldConfiguration from './field-configuration.vue';

export default defineComponent({
	components: { FieldConfiguration },
	props: {
		collection: {
			type: Object as PropType<Collection>,
			required: true,
		},
	},
	emits: ['save', 'toggleAdvanced'],
	setup(props) {
		const { collection } = toRefs(props);

		const { t } = useI18n();

		const fieldDetail = useFieldDetailStore();
		watch(collection, () => fieldDetail.update({ collection: collection.value.collection }), { immediate: true });

		const { interfaces } = getInterfaces();

		const interfacesSorted = computed(() => {
			return orderBy(
				interfaces.value.filter((inter) => !inter.system),
				['order']
			);
		});

		const groups = computed(() => [
			{
				key: 'standard',
				name: t('interface_group_text_and_numbers'),
				interfaces: interfacesSorted.value.filter((inter) => inter.group === 'standard'),
			},
			{
				key: 'selection',
				name: t('interface_group_selection'),
				interfaces: interfacesSorted.value.filter((inter) => inter.group === 'selection'),
			},
			{
				key: 'relational',
				name: t('interface_group_relational'),
				interfaces: interfacesSorted.value.filter((inter) => inter.group === 'relational'),
			},
			{
				key: 'presentation',
				name: t('interface_group_presentation'),
				interfaces: interfacesSorted.value.filter((inter) => inter.group === 'presentation'),
			},
			{
				key: 'group',
				name: t('interface_group_groups'),
				interfaces: interfacesSorted.value.filter((inter) => inter.group === 'group'),
			},
			{
				key: 'other',
				name: t('interface_group_other'),
				interfaces: interfacesSorted.value.filter((inter) => inter.group === 'other'),
			},
		]);

		const chosenInterface = syncFieldDetailStoreProperty('field.meta.interface');

		const configRow = computed(() => {
			if (!chosenInterface.value) return null;

			let indexInGroup: number | null = null;

			groups.value.forEach((group) => {
				const index = group.interfaces.findIndex((inter) => inter.id === chosenInterface.value);
				if (index !== -1) indexInGroup = index;
			});

			if (indexInGroup === null) return null;

			const windowWidth = window.innerWidth;

			let columns = 1;

			if (windowWidth > 400) {
				columns = 2;
			}

			if (windowWidth > 600) {
				columns = 3;
			}

			if (windowWidth > 840) {
				columns = 4;
			}

			return Math.ceil((indexInGroup + 1) / columns) + 1;
		});

		return { t, interfaces, groups, isSVG, syncRefProperty, chosenInterface, configRow, toggleInterface };

		function isSVG(path: string) {
			return path.startsWith('<svg');
		}

		function toggleInterface(id: string) {
			if (chosenInterface.value === id) {
				chosenInterface.value = null;
			} else {
				chosenInterface.value = id;
			}
		}
	},
});
</script>

<style scoped lang="scss">
.content {
	padding: var(--content-padding);
	padding-top: 0;
	padding-bottom: var(--content-padding-bottom);
}

.group h2 {
	margin-bottom: 40px;
	padding-bottom: 2px;
	font-weight: 700;
	border-bottom: var(--border-width) solid var(--border-subdued);
}

.group + .group {
	margin-top: 80px;
}

.grid {
	--columns: 1;

	display: grid;
	grid-template-columns: repeat(var(--columns), 1fr);
	gap: 32px;

	@media (min-width: 400px) {
		--columns: 2;
	}

	@media (min-width: 600px) {
		--columns: 3;
	}

	@media (min-width: 840px) {
		--columns: 4;
	}
}

.interface {
	min-height: 100px;
	overflow: hidden;
	text-align: left;
}

.preview {
	--v-icon-color: var(--background-page);

	display: flex;
	align-items: center;
	justify-content: center;
	width: 160px;
	height: 100px;
	margin-bottom: 8px;
	border: var(--border-width) solid var(--border-subdued);
	border-radius: var(--border-radius);
	transition: var(--fast) var(--transition);
	transition-property: background-color, border-color;
}

.preview img {
	width: 100%;
	height: 100%;
	object-fit: cover;
}

.preview span {
	display: contents;
}

.preview :deep(svg) {
	width: 100%;
	height: 100%;
}

.preview :deep(svg) .glow {
	filter: drop-shadow(0 0 4px var(--primary-50));
}

.preview .fallback {
	--v-icon-color: var(--primary-75);

	display: block;
	padding: 8px 16px;
	background-color: var(--background-page);
	border: 2px solid var(--primary);
	border-radius: var(--border-radius);
	box-shadow: 0 0 8px var(--primary-75);
}

.interface:hover .preview {
	border-color: var(--border-normal);
}

.interface.active .preview {
	background-color: var(--primary-alt);
	border-color: var(--primary);
}

.interface.gray .preview {
	--primary: var(--foreground-subdued);
	--primary-50: var(--foreground-subdued);

	background-color: var(--background-subdued);
}

.interface.gray .preview .fallback {
	--v-icon-color: var(--foreground-subdued);

	box-shadow: 0 0 8px var(--foreground-subdued);
}
</style>
