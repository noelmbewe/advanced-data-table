<script>
  import { getContext, onMount } from "svelte";
  
  // Get Budibase SDK context
  const { styleable, API, Provider } = getContext("sdk");
  const component = getContext("component");
  
  // Component props from schema.json
  export let datasource;
  export let table;
  export let itemsPerPage = 5;
  export let searchable = true;
  export let sortable = true;
  export let selectable = true;
  export let showActions = true;
  export let columns = [];
  export let onRowClick;
  export let onEdit;
  export let onDelete;
  export let onView;
  export let onRefresh;
  
  // Internal state
  let currentPage = 1;
  let searchTerm = "";
  let sortConfig = { key: null, direction: "asc" };
  let selectedRows = new Set();
  let showActionMenu = null;
  let loading = false;
  let error = null;
  let allRows = [];
  let tableSchema = {};
  
  // Reactive data fetching
  $: if (datasource || table) {
    fetchData();
  }
  
  // Get columns from schema or use provided columns
  $: availableColumns = columns.length > 0 ? 
    columns : 
    Object.keys(tableSchema).filter(key => 
      tableSchema[key]?.type !== "link" && 
      tableSchema[key]?.type !== "attachment" &&
      !key.startsWith("_")
    );
  
  // Filter and search data
  $: filteredData = allRows.filter(item => {
    if (!searchTerm) return true;
    return availableColumns.some(col => 
      String(item[col] || "").toLowerCase().includes(searchTerm.toLowerCase())
    );
  });
  
  // Sort data
  $: sortedData = sortConfig.key ? 
    [...filteredData].sort((a, b) => {
      const aValue = a[sortConfig.key];
      const bValue = b[sortConfig.key];
      if (aValue < bValue) return sortConfig.direction === "asc" ? -1 : 1;
      if (aValue > bValue) return sortConfig.direction === "asc" ? 1 : -1;
      return 0;
    }) : filteredData;
  
  // Pagination
  $: totalPages = Math.ceil(sortedData.length / itemsPerPage);
  $: startIndex = (currentPage - 1) * itemsPerPage;
  $: paginatedData = sortedData.slice(startIndex, startIndex + itemsPerPage);
  
  // Context for child components
  $: dataContext = {
    selectedRows: Array.from(selectedRows),
    currentPage,
    totalPages,
    totalRows: sortedData.length,
    loading,
    error,
    refresh: fetchData
  };
  
  
  async function fetchData() {
    if (!datasource && !table) return;
    
    loading = true;
    error = null;
    
    try {
      let response;
      
      if (table) {
        // For table data source, use the correct API method
        const query = {
          tableId: table._id || table,
          limit: 1000 // Adjust as needed
        };
        response = await API.searchTable(query);
        
        // Get table schema - use the table object directly if available
        if (table.schema) {
          tableSchema = table.schema;
        } else {
          // Fallback to API call
          try {
            const tableInfo = await API.fetchTableDefinition(table._id || table);
            tableSchema = tableInfo?.schema || {};
          } catch (schemaError) {
            console.warn("Could not fetch table schema:", schemaError);
            tableSchema = {};
          }
        }
      } else if (datasource) {
        // For other data sources
        response = await API.fetchDatasource(datasource);
      }
      
      // Handle different response formats
      if (response?.rows) {
        allRows = response.rows;
      } else if (response?.data) {
        allRows = response.data;
      } else if (Array.isArray(response)) {
        allRows = response;
      } else {
        allRows = [];
      }
      
      // Reset pagination when data changes
      currentPage = 1;
      selectedRows = new Set();
      
      if (onRefresh) {
        onRefresh({ rows: allRows, totalRows: allRows.length });
      }
      
    } catch (err) {
      console.error("Error fetching data:", err);
      error = err.message || "Failed to fetch data";
      allRows = [];
    } finally {
      loading = false;
    }
  }
  
  // Event handlers
  function handleSort(key) {
    if (!sortable) return;
    sortConfig = {
      key,
      direction: sortConfig.key === key && sortConfig.direction === "asc" ? "desc" : "asc"
    };
  }
  
  function handleRowSelect(rowId) {
    if (!selectable) return;
    const newSelected = new Set(selectedRows);
    if (newSelected.has(rowId)) {
      newSelected.delete(rowId);
    } else {
      newSelected.add(rowId);
    }
    selectedRows = newSelected;
  }
  
  function handleSelectAll() {
    if (!selectable) return;
    if (selectedRows.size === paginatedData.length && paginatedData.length > 0) {
      selectedRows = new Set();
    } else {
      selectedRows = new Set(paginatedData.map(item => item._id || item.id || item._rev));
    }
  }
  
  function handleRowClick(row) {
    if (onRowClick) {
      onRowClick({ row });
    }
  }
  
  function handleEdit(row) {
    if (onEdit) {
      onEdit({ row });
    }
  }
  
  function handleDelete(row) {
    if (onDelete) {
      onDelete({ row });
    }
  }
  
  function handleView(row) {
    if (onView) {
      onView({ row });
    }
  }
  
  function formatValue(value, type) {
    if (value === null || value === undefined) return "";
    
    switch (type) {
      case "datetime":
        return new Date(value).toLocaleDateString();
      case "number":
        return typeof value === "number" ? value.toLocaleString() : value;
      case "boolean":
        return value ? "Yes" : "No";
      case "options":
        return Array.isArray(value) ? value.join(", ") : value;
      default:
        return String(value);
    }
  }
  
  function getColumnType(column) {
    return tableSchema[column]?.type || "text";
  }
  
  function getColumnLabel(column) {
    return tableSchema[column]?.name || column.charAt(0).toUpperCase() + column.slice(1).replace(/([A-Z])/g, ' $1');
  }
  
  function getRowId(row) {
    return row._id || row.id || row._rev || JSON.stringify(row);
  }
  
  // Initialize on mount
  onMount(() => {
    fetchData();
  });
</script>

<div class="data-table-container" use:styleable={$component.styles}>
  <Provider data={dataContext}>
    <!-- Header Section -->
    <div class="table-header">
      <div class="header-content">
        <div class="header-info">
          <h3 class="table-title">Data Table</h3>
          <p class="table-subtitle">
            {#if loading}
              Loading...
            {:else if error}
              Error: {error}
            {:else}
              Showing {paginatedData.length} of {sortedData.length} records
            {/if}
          </p>
        </div>
        
        <div class="header-actions">
          <button 
            class="refresh-button"
            on:click={fetchData}
            disabled={loading}
          >
            🔄 Refresh
          </button>
          
          {#if searchable}
            <div class="search-container">
              <input
                type="text"
                placeholder="Search..."
                bind:value={searchTerm}
                class="search-input"
                disabled={loading}
              />
            </div>
          {/if}
        </div>
      </div>
      
      {#if selectable && selectedRows.size > 0}
        <div class="selection-info">
          {selectedRows.size} row{selectedRows.size === 1 ? '' : 's'} selected
        </div>
      {/if}
    </div>
    
    <!-- Loading State -->
    {#if loading}
      <div class="loading-state">
        <div class="loading-spinner"></div>
        <p>Loading data...</p>
      </div>
    {/if}
    
    <!-- Error State -->
    {#if error && !loading}
      <div class="error-state">
        <p>⚠️ {error}</p>
        <button class="retry-button" on:click={fetchData}>
          Try Again
        </button>
      </div>
    {/if}
    
    <!-- Table Section -->
    {#if !loading && !error}
      <div class="table-wrapper">
        <table class="data-table">
          <thead>
            <tr>
              {#if selectable}
                <th class="select-column">
                  <input
                    type="checkbox"
                    checked={selectedRows.size === paginatedData.length && paginatedData.length > 0}
                    on:change={handleSelectAll}
                  />
                </th>
              {/if}
              
              {#each availableColumns as column}
                <th 
                  class="table-header-cell"
                  class:sortable={sortable}
                  on:click={() => handleSort(column)}
                >
                  <div class="header-content">
                    {getColumnLabel(column)}
                    {#if sortable && sortConfig.key === column}
                      <span class="sort-indicator">
                        {sortConfig.direction === "asc" ? "↑" : "↓"}
                      </span>
                    {/if}
                  </div>
                </th>
              {/each}
              
              {#if showActions}
                <th class="actions-column">Actions</th>
              {/if}
            </tr>
          </thead>
          
          <tbody>
            {#each paginatedData as row, index}
              <tr 
                class="table-row"
                class:selected={selectedRows.has(getRowId(row))}
                on:click={() => handleRowClick(row)}
              >
                {#if selectable}
                  <td class="select-cell">
                    <input
                      type="checkbox"
                      checked={selectedRows.has(getRowId(row))}
                      on:change={() => handleRowSelect(getRowId(row))}
                      on:click|stopPropagation
                    />
                  </td>
                {/if}
                
                {#each availableColumns as column}
                  <td class="table-cell">
                    <div class="cell-content" title={formatValue(row[column], getColumnType(column))}>
                      {formatValue(row[column], getColumnType(column))}
                    </div>
                  </td>
                {/each}
                
                {#if showActions}
                  <td class="actions-cell">
                    <div class="actions-container">
                      <button
                        class="action-button menu-trigger"
                        on:click|stopPropagation={() => 
                          showActionMenu = showActionMenu === getRowId(row) ? null : getRowId(row)
                        }
                      >
                        ⋯
                      </button>
                      
                      {#if showActionMenu === getRowId(row)}
                        <div class="action-menu">
                          {#if onView}
                            <button
                              class="action-menu-item"
                              on:click|stopPropagation={() => handleView(row)}
                            >
                              👁 View
                            </button>
                          {/if}
                          
                          {#if onEdit}
                            <button
                              class="action-menu-item"
                              on:click|stopPropagation={() => handleEdit(row)}
                            >
                              ✏️ Edit
                            </button>
                          {/if}
                          
                          {#if onDelete}
                            <button
                              class="action-menu-item delete"
                              on:click|stopPropagation={() => handleDelete(row)}
                            >
                              🗑 Delete
                            </button>
                          {/if}
                        </div>
                      {/if}
                    </div>
                  </td>
                {/if}
              </tr>
            {/each}
          </tbody>
        </table>
        
        {#if paginatedData.length === 0 && !loading}
          <div class="empty-state">
            <p>
              {searchTerm ? `No results found for "${searchTerm}"` : "No data to display"}
            </p>
            {#if searchTerm}
              <button class="clear-search" on:click={() => searchTerm = ""}>
                Clear search
              </button>
            {/if}
          </div>
        {/if}
      </div>
      
      <!-- Pagination Section -->
      {#if totalPages > 1}
        <div class="pagination-container">
          <div class="pagination-info">
            Page {currentPage} of {totalPages}
          </div>
          
          <div class="pagination-controls">
            <button
              class="pagination-button"
              disabled={currentPage === 1}
              on:click={() => currentPage = Math.max(1, currentPage - 1)}
            >
              Previous
            </button>
            
            {#each Array.from({length: Math.min(5, totalPages)}, (_, i) => {
              if (totalPages <= 5) return i + 1;
              if (currentPage <= 3) return i + 1;
              if (currentPage >= totalPages - 2) return totalPages - 4 + i;
              return currentPage - 2 + i;
            }) as pageNum}
              <button
                class="pagination-button page-number"
                class:active={currentPage === pageNum}
                on:click={() => currentPage = pageNum}
              >
                {pageNum}
              </button>
            {/each}
            
            <button
              class="pagination-button"
              disabled={currentPage === totalPages}
              on:click={() => currentPage = Math.min(totalPages, currentPage + 1)}
            >
              Next
            </button>
          </div>
        </div>
      {/if}
    {/if}
    
    <!-- Slot for child components -->
    <slot />
  </Provider>
</div>

<style>
  .data-table-container {
    background: white;
    border-radius: 8px;
    box-shadow: 0 1px 3px rgba(0, 0, 0, 0.1);
    overflow: hidden;
    font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  }
  
  .table-header {
    padding: 1.5rem;
    background: #f8fafc;
    border-bottom: 1px solid #e2e8f0;
  }
  
  .header-content {
    display: flex;
    justify-content: space-between;
    align-items: center;
    gap: 1rem;
    flex-wrap: wrap;
  }
  
  .header-info {
    display: flex;
    flex-direction: column;
    gap: 0.25rem;
  }
  
  .table-title {
    font-size: 1.25rem;
    font-weight: 600;
    color: #1a202c;
    margin: 0;
  }
  
  .table-subtitle {
    font-size: 0.875rem;
    color: #64748b;
    margin: 0;
  }
  
  .header-actions {
    display: flex;
    align-items: center;
    gap: 1rem;
  }
  
  .refresh-button {
    padding: 0.5rem 1rem;
    background: #f3f4f6;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    cursor: pointer;
    font-size: 0.875rem;
    transition: background-color 0.15s ease;
  }
  
  .refresh-button:hover:not(:disabled) {
    background: #e5e7eb;
  }
  
  .refresh-button:disabled {
    opacity: 0.6;
    cursor: not-allowed;
  }
  
  .search-container {
    display: flex;
    align-items: center;
  }
  
  .search-input {
    padding: 0.5rem 0.75rem;
    border: 1px solid #d1d5db;
    border-radius: 6px;
    font-size: 0.875rem;
    min-width: 200px;
  }
  
  .search-input:focus {
    outline: none;
    border-color: #3b82f6;
    box-shadow: 0 0 0 3px rgba(59, 130, 246, 0.1);
  }
  
  .search-input:disabled {
    background: #f9fafb;
    color: #9ca3af;
  }
  
  .selection-info {
    margin-top: 0.75rem;
    padding: 0.5rem 0.75rem;
    background: #dbeafe;
    color: #1e40af;
    border-radius: 6px;
    font-size: 0.875rem;
  }
  
  .loading-state,
  .error-state {
    display: flex;
    flex-direction: column;
    align-items: center;
    justify-content: center;
    padding: 3rem 1rem;
    color: #6b7280;
  }
  
  .loading-spinner {
    width: 2rem;
    height: 2rem;
    border: 2px solid #f3f4f6;
    border-top: 2px solid #3b82f6;
    border-radius: 50%;
    animation: spin 1s linear infinite;
    margin-bottom: 1rem;
  }
  
  @keyframes spin {
    0% { transform: rotate(0deg); }
    100% { transform: rotate(360deg); }
  }
  
  .error-state {
    color: #dc2626;
  }
  
  .retry-button {
    margin-top: 1rem;
    padding: 0.5rem 1rem;
    background: #dc2626;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
  }
  
  .retry-button:hover {
    background: #b91c1c;
  }
  
  .table-wrapper {
    overflow-x: auto;
  }
  
  .data-table {
    width: 100%;
    border-collapse: collapse;
  }
  
  .table-header-cell {
    background: #f8fafc;
    padding: 0.75rem 1rem;
    text-align: left;
    font-weight: 600;
    color: #374151;
    border-bottom: 1px solid #e5e7eb;
    font-size: 0.875rem;
    white-space: nowrap;
  }
  
  .table-header-cell.sortable {
    cursor: pointer;
    user-select: none;
  }
  
  .table-header-cell.sortable:hover {
    background: #f1f5f9;
  }
  
  .header-content {
    display: flex;
    align-items: center;
    gap: 0.5rem;
  }
  
  .sort-indicator {
    color: #3b82f6;
    font-weight: bold;
  }
  
  .select-column,
  .actions-column {
    width: 60px;
  }
  
  .table-row {
    transition: background-color 0.15s ease;
  }
  
  .table-row:hover {
    background: #f8fafc;
    cursor: pointer;
  }
  
  .table-row.selected {
    background: #eff6ff;
  }
  
  .table-cell,
  .select-cell,
  .actions-cell {
    padding: 0.75rem 1rem;
    border-bottom: 1px solid #f1f5f9;
    font-size: 0.875rem;
    color: #374151;
  }
  
  .cell-content {
    max-width: 200px;
    overflow: hidden;
    text-overflow: ellipsis;
    white-space: nowrap;
  }
  
  .actions-container {
    position: relative;
    display: flex;
    justify-content: center;
  }
  
  .action-button {
    background: none;
    border: none;
    padding: 0.25rem 0.5rem;
    cursor: pointer;
    border-radius: 4px;
    font-size: 1rem;
  }
  
  .action-button:hover {
    background: #f1f5f9;
  }
  
  .action-menu {
    position: absolute;
    top: 100%;
    right: 0;
    background: white;
    border: 1px solid #e5e7eb;
    border-radius: 6px;
    box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
    z-index: 10;
    min-width: 120px;
  }
  
  .action-menu-item {
    display: block;
    width: 100%;
    padding: 0.5rem 0.75rem;
    text-align: left;
    background: none;
    border: none;
    cursor: pointer;
    font-size: 0.875rem;
    color: #374151;
  }
  
  .action-menu-item:hover {
    background: #f3f4f6;
  }
  
  .action-menu-item.delete {
    color: #dc2626;
  }
  
  .action-menu-item.delete:hover {
    background: #fef2f2;
  }
  
  .empty-state {
    text-align: center;
    padding: 3rem 1rem;
    color: #6b7280;
  }
  
  .clear-search {
    margin-top: 0.5rem;
    padding: 0.5rem 1rem;
    background: #3b82f6;
    color: white;
    border: none;
    border-radius: 6px;
    cursor: pointer;
    font-size: 0.875rem;
  }
  
  .clear-search:hover {
    background: #2563eb;
  }
  
  .pagination-container {
    display: flex;
    justify-content: space-between;
    align-items: center;
    padding: 1rem 1.5rem;
    background: #f8fafc;
    border-top: 1px solid #e5e7eb;
  }
  
  .pagination-info {
    font-size: 0.875rem;
    color: #6b7280;
  }
  
  .pagination-controls {
    display: flex;
    gap: 0.25rem;
  }
  
  .pagination-button {
    padding: 0.5rem 0.75rem;
    border: 1px solid #d1d5db;
    background: white;
    color: #374151;
    cursor: pointer;
    border-radius: 4px;
    font-size: 0.875rem;
    transition: all 0.15s ease;
  }
  
  .pagination-button:hover:not(:disabled) {
    background: #f3f4f6;
    border-color: #9ca3af;
  }
  
  .pagination-button:disabled {
    opacity: 0.5;
    cursor: not-allowed;
  }
  
  .pagination-button.page-number {
    min-width: 2.5rem;
  }
  
  .pagination-button.active {
    background: #3b82f6;
    color: white;
    border-color: #3b82f6;
  }
  
  @media (max-width: 768px) {
    .header-content {
      flex-direction: column;
      align-items: stretch;
    }
    
    .header-actions {
      flex-direction: column;
      gap: 0.5rem;
    }
    
    .search-input {
      min-width: auto;
      width: 100%;
    }
    
    .pagination-container {
      flex-direction: column;
      gap: 1rem;
    }
    
    .table-header,
    .table-cell,
    .select-cell,
    .actions-cell {
      padding: 0.5rem;
      font-size: 0.8rem;
    }
    
    .cell-content {
      max-width: 120px;
    }
  }
</style>