## Plugins

Documentation on how to create a new plugin and extend the existing plugins.

## Intro to plugins

Backstage is a single-page application composed of a set of plugins which provide features. Each plugin is treated as a self-contained web app and can include almost any type of content.
Plugins looks like a mini project on its own with a package.json and src folder. Read more about a [Plugin Developement](https://backstage.io/docs/plugins/plugin-development) and [Structure of a Plugin](https://backstage.io/docs/plugins/structure-of-a-plugin).

## Create a Backstage Plugin
First, clone the backstage app repository. Create a new branch from master. For example feature/plugin-example. Make sure you have run **yarn install** and installed dependencies, then run the following on your command line (a shortcut to invoking the [backstage-cli create-plugin][https://backstage.io/docs/cli/commands#create-plugin]) from the root of your project.
```bash
yarn create-plugin
```
![](./pictures/output.png)

This will create a new Backstage plugin based on the ID that was provided. It will be built and added to the Backstage App automatically.

---
**NOTE**
If **yarn start** is already running you should be able to see the default page for your new plugin directly by navigating to **http://localhost:3000/my-plugin**.
---
![](./pictures/output2.png)

You can also serve the plugin in isolation by running **yarn start** in the plugin directory. Or by using the yarn workspace command, for example:
```bash
yarn workspace @backstage/plugin-welcome start # Also supports --check
```
This method of serving the plugin provides quicker iteration speed and a faster startup and hot reloads. It is only meant for local development, and the setup for it can be found inside the plugin's dev/ directory.

## Plugin Customization
In this section, we will describe how you can customize and extend the plugin. The process of customizing the plugin will be described with an example of software catalog plugin.

### Catalog customization
The Backstage software catalog comes with a default CatalogindexPage to filter and find catalog entities.

If you want to change the default index page - such as to add a custom filter to the catalog - you can replace the routing in App.tsx to point to your own customized CatalogIndexPage.

---
**NOTE**

Note: The catalog index page is designed to have a minimal code footprint to support easy customization, however, creating a copy does introduce a possibility of drifting out of date over time. Be sure to check the catalog [CHANGELOG] (https://github.com/backstage/backstage/blob/master/plugins/catalog/CHANGELOG.md) periodically.

---

For example, suppose that I want to allow filtering by a custom annotation added to entities, **company.com/security-tier**. First, I will need to create a **new plugin**. Then, I will copy the code for the default catalog page, create a new component called **CustomCatalogPage** in a new plugin and past that code.

```ts
// imports, etc omitted for brevity. for full source see:
// https://github.com/backstage/backstage/blob/master/plugins/catalog/src/components/CatalogPage/CatalogPage.tsx
export const CustomCatalogPage = () => {
  return (
    <CatalogLayout>
      <Content>
        <ContentHeader title="Components">
          <CreateComponentButton />
          <SupportButton>All your software catalog entities</SupportButton>
        </ContentHeader>
        <div className={styles.contentWrapper}>
          <EntityListProvider>
            <div>
              <EntityKindPicker initialFilter="component" hidden />
              <EntityTypePicker />
              <UserListPicker />
              <EntityTagPicker />
            </div>
            <CatalogTable />
          </EntityListProvider>
        </div>
      </Content>
    </CatalogLayout>
  );
};
```
The **EntityListProvider** shown here provides a list of entities from the catalog-backend, and a way to hook in filters.
Now we are ready to create a new filter that implements the **EntityFilter** interface:

```ts
import { EntityFilter } from '@backstage/plugin-catalog-react';
import { Entity } from '@backstage/catalog-model';

class EntitySecurityTierFilter implements EntityFilter {
  constructor(readonly values: string[]) {}
  filterEntity(entity: Entity): boolean {
    const tier = entity.metadata.annotations?.['company.com/security-tier'];
    return tier !== undefined && this.values.includes(tier);
  }
}
```
The **EntityFilter** interface permits backend filters, which are passed along to the **catalog-backend** - or frontend filters, which are applied after entities are loaded from the backend. 
We will use this filter to extend the default filters in a type-safe way. Let's create the custom filter shape extending the default somewhere alongside this filter:

```ts
export type CustomFilters = DefaultEntityFilters & {
  securityTiers?: EntitySecurityTierFilter;
};
```
To control this filter, we can create a React component that shows checkboxes for the security tiers. This component will make use of the **useEntityListProvider** hook, which accepts this extended filter type as a [generic](https://www.typescriptlang.org/docs/handbook/2/generics.html) parameter:

```tsx
export const EntitySecurityTierPicker = () => {
  // The securityTiers key is recognized due to the CustomFilter generic
  const {
    filters: { securityTiers },
    updateFilters,
  } = useEntityListProvider<CustomFilters>();

  // Toggles the value, depending on whether it's already selected
  function onChange(value: string) {
    const newTiers = securityTiers?.values.includes(value)
      ? securityTiers.values.filter(tier => tier !== value)
      : [...(securityTiers?.values ?? []), value];
    updateFilters({
      securityTiers: newTiers.length
        ? new EntitySecurityTierFilter(newTiers)
        : undefined,
    });
  }

  const tierOptions = ['1', '2', '3'];
  return (
    <FormControl component="fieldset">
      <Typography variant="button">Security Tier</Typography>
      <FormGroup>
        {tierOptions.map(tier => (
          <FormControlLabel
            key={tier}
            control={
              <Checkbox
                checked={securityTiers?.values.includes(tier)}
                onChange={() => onChange(tier)}
              />
            }
            label={`Tier ${tier}`}
          />
        ))}
      </FormGroup>
    </FormControl>
  );
};
```
Now we can add the component to **CustomCatalogPage**:
```tsx
export const CustomCatalogPage = () => {
  return (
    ...
          <EntityListProvider>
            <div>
              <EntityKindPicker initialFilter="component" hidden />
              <EntityTypePicker />
              <UserListPicker />
+              <EntitySecurityTierPicker /> // new component added
              <EntityTagPicker />
            </div>
            <CatalogTable />
          </EntityListProvider>
    ...
};
```
This page itself can be exported as a routable extension in the plugin:

```tsx
export const CustomCatalogIndexPage = myPlugin.provide(
  createRoutableExtension({
    component: () =>
      import('./components/CustomCatalogPage').then(m => m.CustomCatalogPage),
    mountPoint: catalogRouteRef,
  }),
);
```
Finally, we can replace the catalog route in the Backstage application with our new **CustomCatalogIndexpage**
```tsx
# packages/app/src/App.tsx
const routes = (
  <FlatRoutes>
    <Navigate key="/" to="/catalog" />
-    <Route path="/catalog" element={<CatalogIndexPage />} />
+    <Route path="/catalog" element={<CustomCatalogIndexPage />} /> //new route added
```
The same method can be used to customize the default filters with a different interface - for such usage, the generic argument isn't needed since the filter shape remains the same as the default.