[![build status](https://secure.travis-ci.org/reactabular/treebular.svg)](http://travis-ci.org/reactabular/treetabular) [![bitHound Score](https://www.bithound.io/github/reactabular/treetabular/badges/score.svg)](https://www.bithound.io/github/treetabular/sortabular) [![codecov](https://codecov.io/gh/reactabular/treetabular/branch/master/graph/badge.svg)](https://codecov.io/gh/reactabular/treetabular)

# Treetabular - Tree utilities

`treetabular` provides tree helpers for Reactabular. It relies on a flat structure like this:

```javascript
const tree = [
  {
    id: 123,
    name: 'Demo'
  },
  {
    id: 456,
    name: 'Another',
    parent: 123
  },
  {
    id: 789,
    name: 'Yet Another',
    parent: 123
  },
  {
    id: 532,
    name: 'Foobar'
  }
];
```

If there's a `parent` relation, the children must follow their parent right after it.

> You can find suggested default styling for the package at `style.css` in the package root.

## API

```javascript
import * as tree from 'treetabular';

// Or you can cherry-pick
import { filter } from 'treetabular';
import { filter as filterTree } from 'treetabular';
```

### Transformations

**`tree.collapseAll = ({ property = 'showingChildren' }) => (rows) => [<collapsedRow>]`**

Collapses rows by setting `showingChildren` of each row to `false`.

**`tree.expandAll = ({ property = 'showingChildren' }) => (rows) => [<expandedRow>]`**

Expands rows by setting `showingChildren` of each row to `true`.

**`tree.filter = ({ fieldName, parentField = 'parent' }) => (rows) => [<filteredRow>]`**

Filters the given rows using `fieldName`. This is handy if you want only rows that are visible assuming visibility logic has been defined.

### Queries

**`tree.getLevel = ({ index, parentField = 'parent' }) => (rows) => <level>`**

Returns the nesting level of the row at the given `index` within `rows`.

**`tree.getChildren = ({ index, idField = 'id', parentField = 'parent' }) => (rows) => [<child>]`**

Returns children based on given `rows` and `index`. This includes children of children.

**`tree.getImmediateChildren = ({ index, idField = 'id', parentField = 'parent' }) => (rows) => [<child>]`**

Returns immediate children based on given `rows` and `index`.

**`tree.getParents = ({ index, parentField = 'parent' }) => (rows) => [<parent>]`**

Returns parents based on given `rows` and `index`.

**`tree.hasChildren = ({ index, idField = 'id', parentField = 'parent '}) => (rows) => <boolean>`**

Returns a boolean based on whether or not the row at the given `index` has children.

**`tree.search = ({ columns, query, idField = 'id', parentField = 'parent' }) => (rows) => [<searchedRow>]`**

Searches against a tree structure while matching against children too. If children are found, associated parents are returned as well.

> This depends on [resolve.index](http://reactabular.js.org/#/data/resolving)!

**`tree.sort = ({ columns, sortingColumns, strategy, idField = 'id' }) => (rows) => [<sortedRow>]`**

Sorts a tree (packs/unpacks internally to maintain root level sorting).

### Packing

**`tree.pack = ({ parentField = 'parent', childrenField = 'children', idField = 'id' }) => (rows) => [<packedRow>]`**

Packs children inside root level nodes. This is useful with sorting and filtering.

**`tree.unpack = ({ parentField = 'parent', childrenField = 'children', idField = 'id', parent }) => (rows) => [<unpackedRow>]`**

Unpacks children from root level nodes. This is useful with sorting and filtering.

### Drag and Drop

**`tree.moveRows = ({ sourceRowId, targetRowId, retain = [], idField = 'id', parentField = 'parent' }) => (rows) => [<movedRow>]`**

Allows moving tree rows while `retain`ing given fields at their original rows.

### UI

**`tree.toggleChildren = ({ getRows, getShowingChildren, toggleShowingChildren, props, idField, parentField }) => (value, extra) => <React element>`**

Makes it possible to toggle node children through a user interface.

> This depends on [resolve.index](http://reactabular.js.org/#/data/resolving)!

## Example

```jsx
/*
import React from 'react';
import cloneDeep from 'lodash/cloneDeep';
import { compose } from 'redux';
import { Table, resolve } from 'reactabular';
import * as tree from 'reactabular-tree';
import VisibilityToggles from 'reactabular-visibility-toggles';
import * as search from 'searchtabular';
import * as sort from 'sortabular';

import {
  generateParents, generateRows
} from './helpers';
*/

const schema = {
  type: 'object',
  properties: {
    id: {
      type: 'string'
    },
    name: {
      type: 'string'
    },
    age: {
      type: 'integer'
    }
  },
  required: ['id', 'name', 'age']
};

class TreeTable extends React.Component {
  constructor(props) {
    super(props);

    const columns = this.getColumns();
    const rows = resolve.resolve(
      {
        columns,
        method: resolve.index
      }
    )(
      generateParents(generateRows(100, schema))
    );

    this.state = {
      searchColumn: 'all',
      query: {},
      sortingColumns: null,
      rows,
      columns
    };

    this.onExpandAll = this.onExpandAll.bind(this);
    this.onCollapseAll = this.onCollapseAll.bind(this);
    this.onToggleColumn = this.onToggleColumn.bind(this);
  }
  getColumns() {
    const sortable = sort.sort({
      // Point the transform to your rows. React state can work for this purpose
      // but you can use a state manager as well.
      getSortingColumns: () => this.state.sortingColumns || {},

      // The user requested sorting, adjust the sorting state accordingly.
      // This is a good chance to pass the request through a sorter.
      onSort: selectedColumn => {
        const sortingColumns = sort.byColumns({
          sortingColumns: this.state.sortingColumns,
          selectedColumn
        });

        this.setState({ sortingColumns });
      }
    });

    return [
      {
        property: 'name',
        props: {
          style: { width: 200 }
        },
        header: {
          label: 'Name',
          transforms: [sortable]
        },
        cell: {
          formatters: [
            tree.toggleChildren({
              getRows: () => this.state.rows,
              getShowingChildren: ({ rowData }) => rowData.showingChildren,
              toggleShowingChildren: rowIndex => {
                const rows = cloneDeep(this.state.rows);

                rows[rowIndex].showingChildren = !rows[rowIndex].showingChildren;

                this.setState({ rows });
              },
              // Inject custom class name per row here etc.
              props: {}
            })
          ]
        },
        visible: true
      },
      {
        property: 'age',
        props: {
          style: { width: 300 }
        },
        header: {
          label: 'Age',
          transforms: [sortable]
        },
        visible: true
      }
    ];
  }
  render() {
    const {
      searchColumn, columns, sortingColumns, query
    } = this.state;
    const visibleColumns = columns.filter(column => column.visible);
    const rows = compose(
      tree.filter({ fieldName: 'showingChildren' }),
      tree.sort({
        columns,
        sortingColumns
      }),
      tree.search({ columns, query })
    )(this.state.rows);

    return (
      <div>
        <VisibilityToggles
          columns={columns}
          onToggleColumn={this.onToggleColumn}
        />

        <button onClick={this.onExpandAll}>Expand all</button>
        <button onClick={this.onCollapseAll}>Collapse all</button>

        <div className="search-container">
          <span>Search</span>
          <search.Field
            column={searchColumn}
            query={query}
            columns={visibleColumns}
            rows={rows}
            onColumnChange={searchColumn => this.setState({ searchColumn })}
            onChange={query => this.setState({ query })}
          />
        </div>

        <Table.Provider
          className="pure-table pure-table-striped"
          columns={visibleColumns}
        >
          <Table.Header />

          <Table.Body rows={rows} rowKey="id" />
        </Table.Provider>
      </div>
    );
  }
  onExpandAll() {
    this.setState({
      rows: tree.expandAll()(this.state.rows)
    });
  }
  onCollapseAll() {
    this.setState({
      rows: tree.collapseAll()(this.state.rows)
    });
  }
  onToggleColumn(columnIndex) {
    const columns = cloneDeep(this.state.columns);

    columns[columnIndex].visible = !columns[columnIndex].visible;

    this.setState({ columns });
  }
}

<TreeTable />
```

## License

MIT. See LICENSE for details.
