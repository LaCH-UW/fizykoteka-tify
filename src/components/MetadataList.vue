<template>
  <div class="tify-info_metadata">
    <div v-for="(f, index) in apiMetadata" :key="index">
      <template v-if="f.field_name !== 'Tytuł'">
        <h4>{{ f.field_name }}</h4>

        <div class="tify-info_content">
          <span v-html="f.html"></span>
        </div>
      </template>
    </div>
  </div>
</template>

<script>
import httpClient from '@/services/http-client';

const itemMaxLines = 5;
const lineHeight = 24; // [px] value from settings.scss
const maxCharsPerLine = 42; // value empirically determined (and checked against zooming in)

export default {
	props: ['metadata'],
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
					self.apiItem = i.data;
				});
			}
		},
		parseApiItem() {
			const self = this;

			const dates = {
				d1: null,
				d2: null,
			};

			this.apiItem.element_texts.forEach((e) => {
				// console.log(e.element);

				const remove = [41, 46, 50, 64, 65, 40, 70];

				// preparing range dates

				if (e.element.id === 40) dates.d1 = e.text;
				if (e.element.id === 70) dates.d2 = e.text;

				if (!remove.includes(e.element.id)) {
					httpClient.get(e.element.url).then((el) => {
						const m = {
							field_id: el.data.id,
							field_name: e.element.name_translated,
							value: e.text,
							order: Number(el.data.order),
							html: `<a href='/szukaj/p:${el.data.id}/${e.text}'>${e.text}</a>`,
							// hide: _hide,
						};
						self.apiMetadata.push(m);
						self.apiMetadata.sort(
							(a, b) => parseFloat(a.order) - parseFloat(b.order),
						);
						// console.log(self.apiMetadata);
					});
				}
			});

			let r;
			let rhtml;

			if (dates.d1) {
				// parsing dates

				if (dates.d2) {
					r = `${dates.d1} - ${dates.d2}`;
					rhtml = `<a href="/szukaj/p:40/${dates.d1}">${dates.d1}</a> - 
					<a href="/szukaj/p:70/${dates.d2}">${dates.d2}</a>`;
				} else {
					r = dates.d1;
					rhtml = dates.d1;
				}

				const d = {
					field_id: 0, // ??
					field_name: 'Zakres',
					value: r,
					order: 1,
					html: rhtml,
				// hide: _hide,
				};

				self.apiMetadata.push(d);
			}

			// build category

			const colId = this.apiItem.collection.id;
			const colTitle = this.apiItem.collection.collection_title;

			const c = {
				field_id: 0,
				field_name: 'Kolekcja',
				value: this.apiItem.collection.collection_title,
				order: 0,
				html: `<a href='/kolekcja/${colId}'/>${colTitle}</a>`,
				// hide: _hide,
			};

			self.apiMetadata.push(c);
			self.apiMetadata.sort(
				(a, b) => parseFloat(a.order) - parseFloat(b.order),
			);

			// console.log(self.apiMetadata);
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
				const di = this.metadata.indexOf(
					this.metadata.find((x) => x.label === 'Data2'),
				);
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
						let nLines = Math.ceil(
							this.stripHtml(thisValue).length / maxCharsPerLine,
						);
						if (nLines < 1) nLines = 1;
						return linesSum + nLines;
					}, 0);
					const limitHeight = expectedLineNumber > itemMaxLines;
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
