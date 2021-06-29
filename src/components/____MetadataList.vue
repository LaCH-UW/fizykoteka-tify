<template>

	<div class="tify-info_metadata" v-if="null">
		<div v-for="(item, index) in meta" :key="index">

			<template v-if="item.label !== 'Tytuł'">

					<h4>{{item.label}} [{{item.order}}]</h4>

					<div class="tify-info_content">
						<div
						class="tify-info_value"
						:class="{ '-collapsed': infoItems && infoItems[index] && infoItems[index].isCollapsed }"
						:style="infoItems && infoItems[index] && infoItems[index].isCollapsed ? collapsedStyle : null"
						v-bind:key="v" v-for="v in getLabels(item.value)"
						>
						<!-- tutaj values musza isc przez getLabels!!
						w search wyszukiwanie po label
						a moze dodac widok metadane/forma/dfsdfsdf?
						-->

							<a :href="'/szukaj/' + item.label + '/' + v" v-html="v" />

						</div>
					</div>

			</template>

			<template v-if="item.label !== 'Tytuł' && null">
				<h4>
					<div v-bind:key="index" v-for="(label, index) in getLabels(item.label)">
						{{ label|cleanLabel }}
					</div>
				</h4>
				<div class="tify-info_content">
					<div
						class="tify-info_value"
						:class="{ '-collapsed': infoItems && infoItems[index] && infoItems[index].isCollapsed }"
						:style="infoItems && infoItems[index] && infoItems[index].isCollapsed ? collapsedStyle : null"
					>
						<div v-bind:key="i" v-for="(value, i) in getLabels(item.value)">
							<a :href="'/szukaj/' + i">{{value}} / {{i}}</a>
						</div>

					</div>

					<button
						v-if="!infoItems || (infoItems && infoItems[index] && infoItems[index].isInitiallyCollapsed)"
						class="tify-info_toggle"
						@click="infoItems[index].isCollapsed = !infoItems[index].isCollapsed"
					>
						<template v-if="!infoItems || (infoItems && infoItems[index] && infoItems[index].isCollapsed)">
							<icon name="expand_more"/>
							{{ 'Expand'|trans }}
						</template>
						<template v-else>
							<icon name="expand_less"/>
							{{ 'Collapse'|trans }}
						</template>
					</button>
				</div>
			</template>
		</div>
	</div>
</template>

<script>

// <div v-bind:key="value" v-for="value in getLabels(item.value)" v-aname="value" v-html="value"/>

import httpClient from '@/services/http-client';

const itemMaxLines = 5;
const lineHeight = 24; // [px] value from settings.scss
const maxCharsPerLine = 42; // value empirically determined (and checked against zooming in)

export default {
	props: [
		'metadata',
	],
	data() {
		return {
			apiItem: null,
			apiMetadata: [],
			infoItems: null,
			meta: [],
		};
	},
	watch: {
		metadata() {
			this.updateInfoItems();
		},
		apiItem() {
			this.parseApiItem();
		},
	},
	filters: {
		cleanLabel(label) {
			const cleanedLabel = label.replace('_', ' ');
			return cleanedLabel.charAt(0).toUpperCase() + cleanedLabel.substr(1);
		},
	},
	mounted() {
		this.getApiItem();
		this.updateInfoItems();
	},
	methods: {
		getApiItem() {
			// get item data from omeka api

			const u = new URL(this.$root.manifest.related['@id']);
			let id = u.pathname.split('/');
			id = id.pop();

			const apiUri = `${u.protocol}//${u.hostname}/omeka/api/items/${id}`;

			// console.log(apiUri);

			const self = this;

			if (!this.apiItem) {
				httpClient.get(apiUri).then((i) => {
					console.log('request');
					self.apiItem = i.data;
					console.log(self.apiItem);
				});
			}
		},
		parseApiItem() {
			const self = this;

			this.apiItem.element_texts.forEach((e) => {
				// console.log(e.element);
				httpClient.get(e.element.url).then((el) => {
					const m = {
						field_id: el.data.id,
						field_name: e.element.name_translated,
						value: e.text,
						order: Number(el.data.order),
						// hide: _hide,
					};
					self.apiMetadata.push(m);
					self.apiMetadata.sort(
						(a, b) => parseFloat(a.order) - parseFloat(b.order),
					);
					console.log(self.apiMetadata);
				});
			});
		},
		updateInfoItems() {
			// ????

			const itemMaxHeight = itemMaxLines * lineHeight;
			this.collapsedStyle = `max-height: ${itemMaxHeight}px; overflow: hidden`;

			const infoItems = [];
			const { length } = Object.values(this.metadata);

			// building date ranges, removing data2
			if (this.metadata.find((x) => x.label === 'Data2')) {
				const r1 = this.metadata.find((x) => x.label === 'Data');
				const r2 = this.metadata.find((x) => x.label === 'Data2');
				r1.value = `${r1.value} - ${r2.value}`;
				const di = this.metadata.indexOf(this.metadata.find((x) => x.label === 'Data2'));
				this.metadata.splice(di, 1);
			}

			const self = this;

			this.metadata.forEach((m) => {
				// console.log(this.getLabels(m.value));
				// prevent duplicates while watching metadata
				if (!self.meta.find((x) => x.label === m.label)) {
					self.meta.push({
						label: m.label,
						value: m.value,
						order: 1,
						field: 1,
					});
				}
			});

			// uniqueItems = [...new Set(items)]

			this.meta = [...new Set(this.meta)];

			// console.log(this.meta);

			for (let i = 0; i < length; i += 1) {
				if (typeof this.metadata.value !== 'undefined') {
					const item = this.metadata[i];
					const values = this.$root.convertValueToArray(item.value);

					const expectedLineNumber = values.reduce((linesSum, thisValue) => {
						// assuming we need 1 line minimum to display each value
						// and a fixed number of chars fits into each line
						let nLines = Math.ceil(this.stripHtml(thisValue).length / maxCharsPerLine);
						if (nLines < 1) nLines = 1;
						return linesSum + nLines;
					}, 0);
					const limitHeight = (expectedLineNumber > itemMaxLines);
					const infoItem = {
						isCollapsed: limitHeight,
						isInitiallyCollapsed: limitHeight,
					};
					infoItems.push(infoItem);
				}
			}

			this.infoItems = infoItems;
		},
		stripHtml(html) {
			const doc = new DOMParser().parseFromString(html, 'text/html');
			return doc.body.textContent || '';
		},
		getLabels(value) {
			return this.$root.convertValueToArray(value);
		},
	},
};
</script>
