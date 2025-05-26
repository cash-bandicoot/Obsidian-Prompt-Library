```datacorejsx
/**
 * AI Prompts Manager
 * 
 * A Datacore component that manages AI prompts with:
 * - Quick-add functionality
 * - Favorites section with horizontal scroll
 * - Paginated table view of all prompts
 * - Advanced search and filtering
 * - Tagging system with autocomplete
 * - Batch operations
 * - Recently used section
 * - Categories for organization
 * 
 * OPTIMIZED: Single-pass filtering approach for better performance
 */

try {
  // Component Definition
  function AIPromptsManager() {
    // State Management
    const [showForm, setShowForm] = dc.useState(false);
    const [formData, setFormData] = dc.useState({
      title: '',
      description: '',
      tags: '',
      prompt: '',
      favorite: false,
      category: 'General' // Default category
    });
    const [isSubmitting, setIsSubmitting] = dc.useState(false);
    const [error, setError] = dc.useState(null);
    const [success, setSuccess] = dc.useState(null);
    const [searchTerm, setSearchTerm] = dc.useState('');
    const [sortColumn, setSortColumn] = dc.useState('title');
    const [sortDirection, setSortDirection] = dc.useState('asc');
    const [rowsPerPage, setRowsPerPage] = dc.useState(10);
    const [page, setPage] = dc.useState(1);
    const [isNewCategory, setIsNewCategory] = dc.useState(false);
    const [newCategoryName, setNewCategoryName] = dc.useState('');
    const [hoveredCategory, setHoveredCategory] = dc.useState(null);
    const [editingCategory, setEditingCategory] = dc.useState(null);
    const [editCategoryName, setEditCategoryName] = dc.useState('');
    const [localCategories, setLocalCategories] = dc.useState(['General']);
    const [showCategoryManagement, setShowCategoryManagement] = dc.useState(false);
    const [activeTab, setActiveTab] = dc.useState('favorites');
    const [orderedFavorites, setOrderedFavorites] = dc.useState([]);
    const isDarkTheme = document.body.classList.contains('theme-dark');
    const [filterTiming, setFilterTiming] = dc.useState(0); // Added for performance monitoring
    const [categoryColors, setCategoryColors] = dc.useState(() => {
    return JSON.parse(localStorage.getItem('categoryColors') || '{}');
  });
  const [editingCategoryColor, setEditingCategoryColor] = dc.useState(null);
  // don’t reference categoryColors here:
  const [selectedColor, setSelectedColor] = dc.useState('#ffffff');

  // when you click the circle, pull in the saved color:
  const handleOpenColorPicker = (category) => {
    setEditingCategoryColor(category);
    setSelectedColor(categoryColors[category] || '#ffffff');
  };

  const nameChanged = 
    editingCategory 
    && editCategoryName.trim() 
    && editCategoryName.trim() !== editingCategory;

  const colorChanged = 
    editingCategory 
    && selectedColor 
    && selectedColor !== categoryColors[editingCategory];
    

    // Load favorite order from persistent storage if available
    const loadFavoriteOrder = () => {
      try {
        if (window.localStorage) {
          const stored = window.localStorage.getItem('prompt-manager-favorite-order');
          if (stored) {
            return JSON.parse(stored);
          }
        }
      } catch (err) {
        console.error('Failed to load favorite order:', err);
      }
      return null;
    };

    // Save favorite order to persistent storage if available
    const saveFavoriteOrder = (orderedPaths) => {
      try {
        if (window.localStorage) {
          window.localStorage.setItem('prompt-manager-favorite-order', JSON.stringify(orderedPaths));
          console.log('Saved favorite order:', orderedPaths);
        }
      } catch (err) {
        console.error('Failed to save favorite order:', err);
      }
    };

    // Custom hook to debounce user input
    const useDebounce = (value, delay) => {
      const [debouncedValue, setDebouncedValue] = dc.useState(value);
      
      dc.useEffect(() => {
        const handler = setTimeout(() => {
          setDebouncedValue(value);
        }, delay);
        
        return () => clearTimeout(handler);
      }, [value, delay]);
      
      return debouncedValue;
    };

    // Use the hook to create a debounced search term
    const debouncedSearchTerm = useDebounce(searchTerm, 200);

    const [recentlyUsedPrompts, setRecentlyUsedPrompts] = dc.useState(() => {
        try {
            const stored = localStorage.getItem('prompt-manager-recently-used');
            return stored ? JSON.parse(stored) : [];
        } catch {
            return [];
        }
    });

    dc.useEffect(() => {
        try {
            localStorage.setItem(
            'prompt-manager-recently-used',
            JSON.stringify(recentlyUsedPrompts)
            );
        } catch (err) {
            console.error('Could not save recently used prompts:', err);
        }
    }, [recentlyUsedPrompts]);

    const markAsRecentlyUsedPrompt = (prompt) => {
        if (!prompt?.$path) return;
        
        const entry = {
            // Use consistent property name
            $path: prompt.$path, // Change from path to $path for consistency
            path: prompt.$path,  // Keep this for backward compatibility
            title: prompt.title,
            description: prompt.description || '',
            prompt: typeof prompt.prompt === 'string'
            ? prompt.prompt
            : JSON.stringify(prompt.prompt),
            timestamp: new Date().toISOString(),
            category: prompt.category || 'General',
            tags: prompt.tags || [],
            favorite: prompt.favorite || false
        };
        
        setRecentlyUsedPrompts(prev =>
            [entry, ...prev.filter(item => item.$path !== entry.$path && item.path !== entry.$path)].slice(0, 15)
        );
        };

    // Set up scroll event listeners for favorites container
    dc.useEffect(() => {
        const checkScrollability = () => {
            const container = document.getElementById('favorites-scroll-container');
            if (!container) return;
            const leftArrow  = document.getElementById('scroll-arrow-left');
            const rightArrow = document.getElementById('scroll-arrow-right');
            if (!leftArrow || !rightArrow) return;

            const atStart = container.scrollLeft <= 10;
            const atEnd   = container.scrollLeft + container.clientWidth >= container.scrollWidth - 10;

            leftArrow.style.display  = atStart ? 'none' : '';
            rightArrow.style.display = atEnd   ? 'none' : '';
        };

        const container = document.getElementById('favorites-scroll-container');
        if (!container) return;

        // initial
        checkScrollability();
        // attach
        container.addEventListener('scroll',  checkScrollability);
        window.addEventListener('resize',     checkScrollability);

        return () => {
            container.removeEventListener('scroll', checkScrollability);
            window.removeEventListener('resize',    checkScrollability);
        };
    }, [
        activeTab,
        orderedFavorites.length,
        recentlyUsedPrompts.length
    ]);
        
    // Advanced search state
    const [selectedTags, setSelectedTags] = dc.useState([]);
    const [searchFilters, setSearchFilters] = dc.useState({
      onlyFavorites: false,
      dateRange: { from: null, to: null },
      contentType: 'all', // 'all', 'title', 'description', 'prompt'
      category: 'all' // Filter by category
    });
    const [showFilters, setShowFilters] = dc.useState(false);
    const hasDateFilter =
      searchFilters.dateRange &&
      searchFilters.dateRange.from &&
      searchFilters.dateRange.to;
    const [recentSearches, setRecentSearches] = dc.useState([]);
    
    // Batch operations state
    const [selectedPrompts, setSelectedPrompts] = dc.useState([]);
    const [showBatchActions, setShowBatchActions] = dc.useState(false);
    const [batchTagInput, setBatchTagInput] = dc.useState('');
    const [showBatchConfirm, setShowBatchConfirm] = dc.useState(false);
    const [batchAction, setBatchAction] = dc.useState('');

    // Tag autocomplete state
    const [tagInput, setTagInput] = dc.useState('');
    const [tagSuggestions, setTagSuggestions] = dc.useState([]);
    const [showTagSuggestions, setShowTagSuggestions] = dc.useState(false);

    // ─── Drag-and-Drop State Hooks ───
    const [draggedIndex, setDraggedIndex] = dc.useState(null);
    const [dragOverIndex, setDragOverIndex] = dc.useState(null);

    // Component Styles
    const styles = {
      container: {
        padding: '20px',
        maxWidth: '100%',
        margin: '0 auto',
        fontFamily: 'var(--font-interface)'
      },
      scrollGrid: {
        display: 'grid',
        gridAutoFlow: 'column',               // fill top→down then next column
        gridTemplateRows: 'repeat(2, auto)',  // exactly 2 rows
        gap: '16px',                          // adjust card spacing
        overflowX: 'auto',                    // scroll horizontally
        overflowY: 'hidden',                  // hide any vertical overflow
        padding: '8px 0'
      },
      header: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        marginBottom: '20px'
      },
      title: {
        fontSize: '24px',
        fontWeight: 'bold',
        margin: 0
      },
      addButton: {
        display: 'flex',
        alignItems: 'center',
        gap: '6px',
        padding: '8px 16px',
        backgroundColor: isDarkTheme ? '#4A5568' : '#3182CE',
        color: 'white',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontWeight: '500',
        transition: 'background-color 0.2s'
      },
      form: {
        backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC',
        padding: '20px',
        borderRadius: '8px',
        marginBottom: '20px',
        boxShadow: '0 2px 6px rgba(0, 0, 0, 0.1)'
      },
      formGroup: {
        marginBottom: '15px'
      },
      label: {
        display: 'block',
        marginBottom: '6px',
        fontWeight: '500'
      },
      input: {
        width: '100%',
        padding: '4px 5px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit'
      },
      textarea: {
        width: '100%',
        padding: '8px 10px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        minHeight: '100px',
        fontFamily: 'monospace'
      },
      checkbox: {
        display: 'flex',
        alignItems: 'center',
        gap: '8px'
      },
      formActions: {
        display: 'flex',
        justifyContent: 'flex-end',
        gap: '10px',
        marginTop: '20px'
      },
      cancelButton: {
        padding: '8px 16px',
        backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer'
      },
      submitButton: {
        padding: '8px 16px',
        backgroundColor: isDarkTheme ? '#4C51BF' : '#4299E1',
        color: 'white',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontWeight: '500'
      },
      errorMessage: {
        backgroundColor: isDarkTheme ? '#742A2A' : '#FED7D7',
        color: isDarkTheme ? '#FEB2B2' : '#9B2C2C',
        padding: '10px 15px',
        borderRadius: '4px',
        marginBottom: '20px'
      },
      successMessage: {
        backgroundColor: isDarkTheme ? '#22543D' : '#C6F6D5',
        color: isDarkTheme ? '#9AE6B4' : '#276749',
        padding: '10px 15px',
        borderRadius: '4px',
        marginBottom: '20px'
      },
      sectionTitle: {
        fontSize: '20px',
        fontWeight: '600',
        marginBottom: '16px',
        display: 'flex',
        alignItems: 'center',
        gap: '8px',
        position: 'relative',
        paddingBottom: '8px',
        borderBottom: `2px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`
      },
      // Updated favorites container style
      favoritesContainer: {
        marginBottom: '30px'
      },

      // Updated favorites scroll style to ensure proper alignment
      favoritesScroll: {
        display: 'grid',
        gridAutoFlow: 'column',              // stack items top→down, then next column
        gridTemplateRows: 'repeat(2, auto)', // exactly 2 rows
        gap: '16px',                         // spacing between cards
        overflowX: 'auto',                   // horizontal scroll
        overflowY: 'hidden',                 // hide any extra rows
        padding: '8px 50px',                 // adjust as needed
        maxWidth: '100%',
        scrollBehavior: 'smooth',
        scrollbarWidth: 'thin',
        scrollbarColor: isDarkTheme
          ? '#4A5568 #1A202C'
          : '#CBD5E0 #F7FAFC',
        '&::-webkit-scrollbar': {
          height: '5px'
        },
        '&::-webkit-scrollbar-track': {
          backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',
          borderRadius: '4px'
        },
        '&::-webkit-scrollbar-thumb': {
          backgroundColor: isDarkTheme ? '#4A5568' : '#CBD5E0',
          borderRadius: '4px',
          '&:hover': {
            backgroundColor: isDarkTheme ? '#718096' : '#A0AEC0'
          }
        },
        alignItems: 'flex-start', // top-align cards rather than stretch
      },

      // Update the favorites section header style
      favoritesHeader: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        marginBottom: '16px',
        borderBottom: `2px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        paddingBottom: '8px',
      },

      // Update the container for the scrollable content
      favoritesScrollContainer: {
        position: 'relative',
        marginBottom: '20px',
        backgroundColor: isDarkTheme ? '#1E2A3B10' : '#F7FAFC50',
        borderRadius: '12px',
        padding: '0px 40px', // Increased vertical padding
        minHeight: '450px', // Ensure minimum height to accommodate cards
      },
      card: {
        minWidth: '280px',
        maxWidth: '280px',
        height: '380px', // Increased from 360px to 380px to provide more space
        backgroundColor: isDarkTheme ? '#2D3748' : 'white',
        borderRadius: '8px',
        padding: '24px',
        boxShadow: isDarkTheme 
          ? '0 4px 12px rgba(0, 0, 0, 0.3), 0 1px 3px rgba(0, 0, 0, 0.2)' 
          : '0 4px 12px rgba(0, 0, 0, 0.1), 0 1px 3px rgba(0, 0, 0, 0.05)',
        display: 'flex',
        flexDirection: 'column',
        flex: '0 0 auto',
        position: 'relative',
        transition: 'transform 0.2s ease, box-shadow 0.2s ease, opacity 0.2s ease',
        margin: '2px',
        cursor: 'grab',
        userSelect: 'none'
      },
      categoryBadge: {
        position: 'absolute',
        bottom: '10px',
        right: '10px',
        padding: '4px 8px',
        borderRadius: '4px',
        fontSize: '11px',
        fontWeight: '600',
        textTransform: 'uppercase',
        letterSpacing: '0.5px',
        boxShadow: '0 1px 2px rgba(0,0,0,0.1)',
        /* allow full display */
        whiteSpace: 'normal',
        overflow: 'visible',
        textOverflow: 'clip',
        zIndex: 5,
        opacity: 0.9,
        transition: 'opacity 0.2s ease',
        '&:hover': {
          opacity: 1
        }
      },
      cardDragOver: {
        // outline the drop target in blue
        outline: '2px solid #3182CE',
        // pull the outline inside so it doesn't shift layout
        outlineOffset: '-2px'
      },
      // New styles for card content sections with fixed heights
      cardContentWrapper: {
        display: 'flex',
        flexDirection: 'column',
        flex: 1,
        overflow: 'hidden',
        position: 'relative',
        paddingTop: '15px', // Add some space from the drag handle
      },
      cardTitleSection: {
        height: '40px', // Keep the same
        marginBottom: '10px',
        paddingRight: '8px',
        paddingLeft: '8px',
        display: 'flex',
        alignItems: 'center'
      },
      cardTitle: {
        fontSize: '18px',
        fontWeight: '600',
        width: '100%',
        whiteSpace: 'nowrap',
        overflow: 'hidden',
        textOverflow: 'ellipsis',
        color: isDarkTheme ? '#E2E8F0' : '#2D3748',
        letterSpacing: '0.01em',             // Subtle letter spacing for better readability
        fontFamily: "'Inter', 'SF Pro Display', -apple-system, BlinkMacSystemFont, 'Segoe UI', Roboto, sans-serif",
        padding: '0 2px',                    // Small horizontal padding
        textShadow: isDarkTheme ? '0 1px 2px rgba(0,0,0,0.3)' : 'none', // Shadow for dark mode
        borderBottom: `2px solid ${isDarkTheme ? '#4C51BF30' : '#4299E120'}`, // Subtle accent border
        paddingBottom: '4px',                // Space for the border
        transition: 'color 0.2s ease',       // Smooth color transition on hover
      },
      cardDescriptionSection: {
        height: '75px', // Keep the same
        marginBottom: '6px',
        overflow: 'hidden',
      },
      cardDescription: {
        fontSize: '14px',
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        display: '-webkit-box',
        WebkitLineClamp: 3, 
        WebkitBoxOrient: 'vertical',
        overflow: 'hidden',
        textOverflow: 'ellipsis',
        lineHeight: '1.4',
        margin: 0,
        padding: '0px 5px',
        letterSpacing: '0.01em',  
        fontWeight: '400', 
        transition: 'all 0.2s ease', 
      },
      cardPromptPreviewSection: {
        height: '110px',
        marginBottom: '15px',
        backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',
        padding: '10px',
        borderRadius: '6px',
        overflow: 'hidden',
        position: 'relative',
      },
      cardPromptPreview: {
        fontFamily: "'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, Monaco, 'Courier New', monospace",
        fontSize: '12px',
        color: isDarkTheme ? '#CBD5E0' : '#4A5568',
        whiteSpace: 'pre-wrap',
        wordBreak: 'break-word',
        height: '100%',
        overflow: 'hidden',
        letterSpacing: '0.05em',  // Slightly increased letter spacing for better readability
        lineHeight: '1.4',        // Consistent line height
        backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',  // Subtle background color
        padding: '2px 4px',       // Small padding within the container
        borderRadius: '3px',      // Rounded corners
      },
      cardTagsSection: {
        height: '45px',
        overflow: 'hidden',
        marginBottom: '15px',
        position: 'relative',
      },
      cardTags: {
        display: 'flex',
        flexWrap: 'nowrap', // Prevent wrapping
        gap: '0.5rem',
        overflow: 'auto',
        scrollbarWidth: 'none', // Firefox
        height: '30px', // Fixed height to prevent layout shift
        padding: '2px 0', // Small padding for breathability
        '-webkit-overflow-scrolling': 'touch', // Better touch scrolling
        '&::-webkit-scrollbar': {
          display: 'none', // Hide scrollbar in Chrome/Safari
        },
      },
      cardActionsSection: {
        minHeight: '50px', // Increased from 40px for more space
        marginTop: 'auto',
        paddingTop: '10px', // Add padding to separate from content above
        display: 'flex', // Ensure flexbox layout
        alignItems: 'center', // Center buttons vertically
        justifyContent: 'flex-start', // Align to start
        position: 'relative', // Enable positioning
        bottom: '0', // Anchor to bottom
        width: '100%' // Full width
      },
      tag: {
        fontSize: '11px',
        padding: '3px 8px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#F1F5F9', // Updated color
        color: isDarkTheme ? '#90CDF4' : '#475569', // Updated color
        borderRadius: '12px',
        display: 'inline-flex',
        alignItems: 'center',
        fontWeight: '500',
        whiteSpace: 'nowrap', // Prevent internal breaking
        boxShadow: isDarkTheme ? '0 1px 2px rgba(0,0,0,0.3)' : '0 1px 2px rgba(0,0,0,0.1)',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        transition: 'all 0.2s ease',
        cursor: 'pointer',
        flex: '0 0 auto', // Don't grow or shrink
        '&:hover': {
          backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
          transform: 'translateY(-1px)'
        }
      },
      cardActions: {
        display: 'flex',
        alignItems: 'center',
        gap: '8px',
        width: '100%'
      },
      cardButton: {
        padding: '8px 10px',
        backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',
        color: isDarkTheme ? '#A0AEC0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '14px',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        gap: '4px',
        transition: 'all 0.2s ease',
        width: '40px',
        height: '40px',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#2D3748' : '#EDF2F7',
          color: isDarkTheme ? '#E2E8F0' : '#2D3748',
          transform: 'scale(1.05)'
        }
      },
      dragHandle: {
        position: 'absolute',
        top: '10px',
        left: '10px',
        fontSize: '16px',
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        cursor: 'grab',
        zIndex: 2,
        transition: 'color 0.2s ease'
      },
      dragHandleActive: {
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        cursor: 'grabbing'
      },
      favoriteIndicator: {
        position: 'absolute',
        top: '10px',
        right: '10px',
        fontSize: '18px',
        color: isDarkTheme ? '#F6E05E' : '#F6AD55',
        filter: 'drop-shadow(0 1px 2px rgba(0,0,0,0.25))',
        zIndex: 2,
        transition: 'transform 0.3s ease'
      },
      tableContainer: {
        marginTop: '0px',            
        overflowX: 'auto',
        backgroundColor: isDarkTheme
          ? '#1F2937'
          : '#FFFFFF',                  
        borderRadius: '8px',          
        boxShadow: '0 1px 2px rgba(0,0,0,0.05)', 
        position: 'relative', // Added for proper positioning context
        maxHeight: '70vh',   // Optional - prevents table from becoming too large
        overflowY: 'auto',   // Allows scrolling when table is tall
      },
      table: {
        width: '100%',
        borderCollapse: 'separate',
        borderSpacing: 0,
      },
      tableHead: {
        backgroundColor: isDarkTheme ? '#2D3748' : '#EDF2F7',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        position: 'sticky',
        top: 0,
        zIndex: 10,
        boxShadow: '0 1px 2px rgba(0, 0, 0, 0.1)'
      },
      tableHeader: {
        backgroundColor: isDarkTheme
          ? '#111827'
          : '#F9FAFB',                  // very light gray
        color: isDarkTheme
          ? '#E5E7EB'
          : '#374151',                  // charcoal text
        fontWeight: 600,
        fontSize: '14px',
        textAlign: 'left',
        padding: '16px',
      },
      tableRow: (index) => ({
        transition: 'background-color 0.2s ease',
        backgroundColor: index % 2 === 1 
          ? (isDarkTheme ? 'rgba(255,255,255,0.03)' : 'rgba(0,0,0,0.03)')
          : 'transparent',
        position: 'relative',
      }),
      tableRowHover: {
        backgroundColor: isDarkTheme
          ? 'rgba(66, 153, 225, 0.1)'  // Slight blue tint in dark mode
          : 'rgba(66, 153, 225, 0.05)', // Slight blue tint in light mode
      },
      tableCell: {
        padding: '16px',
        border: 'none',
        color: isDarkTheme
          ? '#D1D5DB'
          : '#4B5563',
      },
      tableTagContainer: {
        display: 'flex',
        flexWrap: 'nowrap', // Prevent wrapping
        gap: '0.5rem',
        overflow: 'auto',
        scrollbarWidth: 'none', // Firefox
        height: '28px', // Fixed height to prevent layout shift
        overflowY: 'hidden', // Ensure no vertical scrolling
        '-webkit-overflow-scrolling': 'touch',
        '&::-webkit-scrollbar': {
          display: 'none',
        },
        padding: '2px 0', // Small padding
      },
      tableTag: {
        fontSize: '12px',
        padding: '3px 8px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#F1F5F9', // Updated color
        color: isDarkTheme ? '#90CDF4' : '#475569', // Updated color
        borderRadius: '12px',
        display: 'inline-flex',
        alignItems: 'center',
        fontWeight: '500',
        whiteSpace: 'nowrap',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        cursor: 'pointer',
        flex: '0 0 auto', // Don't grow or shrink
      },
      tableActions: {
        display: 'flex',
        gap: '8px',
        justifyContent: 'flex-end',
        alignItems: 'center',
      },
      actionButton: {
        padding: '6px 9px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC',
        color: isDarkTheme ? '#A0AEC0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '14px',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
          transform: 'translateY(-1px)'
        }
      },
      pagination: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        marginTop: '20px'
      },
      paginationInfo: {
        fontSize: '14px',
        color: isDarkTheme ? '#A0AEC0' : '#718096'
      },
      paginationButtons: {
        display: 'flex',
        gap: '5px'
      },
      pageButton: {
        padding: '6px 10px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#E2E8F0',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer'
      },
      activePageButton: {
        backgroundColor: isDarkTheme ? '#4C51BF' : '#3182CE',
        color: 'white'
      },
      filterPanel: {
        backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC',
        padding: '15px',
        borderRadius: '8px',
        marginBottom: '15px',
        boxShadow: '0 2px 6px rgba(0, 0, 0, 0.1)',
        animation: 'fadeIn 0.2s ease-in-out'
      },
      filterSection: {
        marginBottom: '15px'
      },
      filterSectionTitle: {
        fontSize: '14px',
        fontWeight: '600',
        marginBottom: '8px',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568'
      },
      tagFilterContainer: {
        display: 'flex',
        flexWrap: 'wrap',
        gap: '8px',
        marginBottom: '15px'
      },
      tagFilter: {
        fontSize: '13px',
        padding: '4px 10px',
        borderRadius: '16px',
        backgroundColor: isDarkTheme ? '#4A5568' : '#EBF8FF',
        color: isDarkTheme ? '#E2E8F0' : '#2C5282',
        cursor: 'pointer',
        transition: 'all 0.2s ease',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#BEE3F8'}`,
        '&:hover': {
          backgroundColor: isDarkTheme ? '#2D3748' : '#BEE3F8'
        }
      },
      tagFilterActive: {
        backgroundColor: isDarkTheme ? '#553C9A' : '#B794F4',
        color: isDarkTheme ? '#E9D8FD' : '#44337A',
        border: `1px solid ${isDarkTheme ? '#6B46C1' : '#9F7AEA'}`
      },
      filterControls: {
        display: 'flex',
        justifyContent: 'space-between',
        gap: '10px',
        marginTop: '15px'
      },
      filterSwitch: {
        display: 'flex',
        alignItems: 'center',
        gap: '8px',
        fontSize: '13px',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568'
      },
      dateContainer: {
        display: 'flex',
        gap: '8px',
        alignItems: 'center'
      },
      filterLabel: {
        fontSize: '13px',
        color: isDarkTheme ? '#A0AEC0' : '#718096'
      },
      filterDateInput: {
        padding: '6px',
        width: '120px',
        fontSize: '13px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit'
      },
      filterSelect: {
        padding: '6px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        fontSize: '13px'
      },
      clearFiltersButton: {
        padding: '6px 12px',
        backgroundColor: isDarkTheme ? '#1A202C' : '#E2E8F0',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '13px'
      },
      filterToggle: {
        display: 'flex',
        alignItems: 'center',
        gap: '5px',
        padding: '6px 12px',
        backgroundColor: 'transparent',
        color: isDarkTheme ? '#A0AEC0' : '#4A5568',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '13px',
        transition: 'all 0.2s ease'
      },
      recentSearchesContainer: {
        marginTop: '8px',
        display: 'flex',
        flexWrap: 'wrap',
        gap: '8px'
      },
      recentSearch: {
        padding: '4px 10px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC',
        color: isDarkTheme ? '#A0AEC0' : '#4A5568',
        borderRadius: '16px',
        fontSize: '12px',
        cursor: 'pointer',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        display: 'flex',
        alignItems: 'center',
        gap: '4px'
      },
      searchResultStats: {
        fontSize: '14px',
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        marginBottom: '10px'
      },
      searchContainer: {
        position: 'relative',
        display: 'flex',
        gap: '10px',
        flex: 1
      },
      searchInput: {
        padding: '8px 10px',
        paddingRight: '34px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        flex: 1
      },
      searchIcon: {
        position: 'absolute',
        right: '10px',
        top: '50%',
        transform: 'translateY(-50%)',
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        fontSize: '16px'
      },
      activeFiltersContainer: {
        display: 'flex',
        flexWrap: 'wrap',
        gap: '8px',
        marginTop: '8px',
        marginBottom: '10px'
      },
      activeFilter: {
        padding: '4px 10px',
        backgroundColor: isDarkTheme ? '#1A202C' : '#EBF8FF',
        color: isDarkTheme ? '#A0AEC0' : '#4A5568',
        borderRadius: '16px',
        fontSize: '12px',
        display: 'flex',
        alignItems: 'center',
        gap: '6px'
      },
      activeFilterLabel: {
        fontWeight: '600',
        color: isDarkTheme ? '#E2E8F0' : '#2D3748'
      },
      activeFilterRemove: {
        cursor: 'pointer',
        fontSize: '16px',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        marginLeft: '4px'
      },
      emptyState: {
        padding: '40px 20px',
        textAlign: 'center',
        backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC',
        borderRadius: '8px',
        color: isDarkTheme ? '#A0AEC0' : '#718096'
      },
      batchActionBar: {
        backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC',
        padding: '10px 15px',
        borderRadius: '8px',
        boxShadow: '0 2px 6px rgba(0, 0, 0, 0.1)',
        marginBottom: '15px',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'space-between',
        gap: '10px',
        animation: 'slideInUp 0.3s ease'
      },
      batchActionGroup: {
        display: 'flex',
        alignItems: 'center',
        gap: '8px'
      },
      batchActionButton: {
        padding: '6px 12px',
        backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '13px',
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#718096' : '#CBD5E0'
        }
      },
      batchDeleteButton: {
        backgroundColor: isDarkTheme ? '#742A2A' : '#FED7D7',
        color: isDarkTheme ? '#FEB2B2' : '#9B2C2C',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#9B2C2C' : '#FEB2B2'
        }
      },
      batchTagInput: {
        padding: '6px 10px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        fontSize: '13px',
        width: '150px'
      },
      batchConfirmOverlay: {
        position: 'fixed',
        top: 0,
        left: 0,
        right: 0,
        bottom: 0,
        backgroundColor: 'rgba(0, 0, 0, 0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 100
      },
      batchConfirmDialog: {
        backgroundColor: isDarkTheme ? '#2D3748' : 'white',
        borderRadius: '8px',
        padding: '20px',
        width: '400px',
        boxShadow: '0 4px 20px rgba(0, 0, 0, 0.3)',
        animation: 'fadeIn 0.2s ease'
      },
      batchConfirmTitle: {
        fontSize: '18px',
        fontWeight: '600',
        marginBottom: '15px',
        color: isDarkTheme ? '#E2E8F0' : '#2D3748'
      },
      batchConfirmText: {
        fontSize: '14px',
        marginBottom: '20px',
        color: isDarkTheme ? '#A0AEC0' : '#718096'
      },
      batchConfirmButtons: {
        display: 'flex',
        justifyContent: 'flex-end',
        gap: '10px'
      },
      batchCancelButton: {
        padding: '8px 16px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#E2E8F0',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '14px'
      },
      batchConfirmButton: {
        padding: '8px 16px',
        backgroundColor: isDarkTheme ? '#742A2A' : '#FED7D7',
        color: isDarkTheme ? '#FEB2B2' : '#9B2C2C',
        border: 'none',
        borderRadius: '4px',
        cursor: 'pointer',
        fontSize: '14px',
        fontWeight: '600'
      },
      categorySection: {
        marginBottom: '20px'
      },
      categoryTitle: {
        fontSize: '16px',
        fontWeight: '600',
        marginBottom: '10px',
        color: isDarkTheme ? '#E2E8F0' : '#2D3748',
        display: 'flex',
        alignItems: 'center',
        gap: '8px'
      },
      categorySelect: {
        padding: '6px 10px',
        borderRadius: '4px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        fontSize: '13px',
        marginLeft: '10px'
      },
      recentlyUsedSection: {
        marginBottom: '30px'
      },
      selectAllCheckbox: {
        width: '18px',
        height: '18px',
        cursor: 'pointer'
      },
      scrollIndicatorLeft: {
        position: 'absolute',
        left: '8px',
        top: '50%',
        transform: 'translateY(-50%)',
        width: '40px',
        height: '40px',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 2,
        cursor: 'pointer',
        backgroundColor: isDarkTheme ? 'rgba(26, 32, 44, 0.8)' : 'rgba(247, 250, 252, 0.8)',
        borderRadius: '50%',
        boxShadow: '0 2px 6px rgba(0, 0, 0, 0.15)',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? 'rgba(45, 55, 72, 0.9)' : 'rgba(237, 242, 247, 0.9)',
          transform: 'translateY(-50%) scale(1.05)'
        }
      },
      scrollIndicatorRight: {
        position: 'absolute',
        right: '8px',
        top: '50%',
        transform: 'translateY(-50%)',
        width: '40px',
        height: '40px',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 2,
        cursor: 'pointer',
        backgroundColor: isDarkTheme ? 'rgba(26, 32, 44, 0.8)' : 'rgba(247, 250, 252, 0.8)',
        borderRadius: '50%',
        boxShadow: '0 2px 6px rgba(0, 0, 0, 0.15)',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? 'rgba(45, 55, 72, 0.9)' : 'rgba(237, 242, 247, 0.9)',
          transform: 'translateY(-50%) scale(1.05)'
        }
      },
      scrollArrow: {
        fontSize: '18px',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        width: '100%',
        height: '100%',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center'
      },
      tagSuggestionContainer: {
        position: 'absolute',
        top: '100%',
        left: 0,
        right: 0,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        borderRadius: '4px',
        marginTop: '2px',
        maxHeight: '150px',
        overflowY: 'auto',
        zIndex: 30,
        boxShadow: '0 4px 8px rgba(0, 0, 0, 0.1)'
      },
      tagSuggestionItem: {
        padding: '8px 10px',
        cursor: 'pointer',
        borderBottom: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        backgroundColor: isDarkTheme ? '#1A202C' : 'white',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#2D3748' : '#F7FAFC'
        }
      },
      tagSuggestionItemLast: {
        borderBottom: 'none'
      },
      tagsContainer: {
        display: 'flex',
        flexWrap: 'wrap',
        gap: '5px',
        marginTop: '5px'
      },
      formTag: {
        padding: '2px 8px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#EBF8FF',
        color: isDarkTheme ? '#90CDF4' : '#2C5282',
        borderRadius: '12px',
        display: 'flex',
        alignItems: 'center',
        gap: '5px',
        fontSize: '12px'
      },
      formTagRemove: {
        cursor: 'pointer',
        fontWeight: 'bold'
      },
      emptyStateText: {
        marginBottom: '16px',
        fontSize: '16px',
        color: isDarkTheme ? '#A0AEC0' : '#718096'
      }
    };

    const renderCategorySelect = () => {
      const categoryStyles = {
        categoryContainer: {
          position: 'relative',
          marginBottom: '15px'
        },
        newCategoryInput: {
          width: '100%',
          padding: '6px 8px',
          marginTop: '8px',
          borderRadius: '4px',
          border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
          backgroundColor: isDarkTheme ? '#1A202C' : 'white',
          color: isDarkTheme ? '#E2E8F0' : 'inherit'
        },
        newCategoryActions: {
          display: 'flex',
          marginTop: '8px',
          gap: '8px'
        },
        categoryButton: {
          padding: '4px 8px',
          backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
          color: isDarkTheme ? '#E2E8F0' : '#4A5568',
          border: 'none',
          borderRadius: '4px',
          cursor: 'pointer',
          fontSize: '13px'
        },
        createButton: {
          backgroundColor: isDarkTheme ? '#4C51BF' : '#4299E1',
          color: 'white',
        },
        manageButton: {
          display: 'inline-flex',
          alignItems: 'center',
          gap: '4px',
          padding: '4px 8px',
          marginTop: '8px',
          backgroundColor: isDarkTheme ? '#2D3748' : '#E2E8F0',
          color: isDarkTheme ? '#A0AEC0' : '#4A5568',
          border: 'none',
          borderRadius: '4px',
          cursor: 'pointer',
          fontSize: '13px',
          transition: 'all 0.2s ease'
        },
        optionWithActions: {
          display: 'flex',
          justifyContent: 'space-between',
          alignItems: 'center',
          padding: '4px 8px',
          cursor: 'default',
          borderRadius: '4px',
          marginBottom: '2px'
        },
        categoryActions: {
          display: 'flex',
          gap: '4px'
        },
        categoryActionButton: {
          backgroundColor: 'transparent',
          border: 'none',
          cursor: 'pointer',
          padding: '2px 4px',
          fontSize: '12px',
          color: isDarkTheme ? '#A0AEC0' : '#718096',
          borderRadius: '3px'
        },
        categoryActionButtonHover: {
          backgroundColor: isDarkTheme ? '#2D3748' : '#EBF8FF',
          color: isDarkTheme ? '#E2E8F0' : '#3182CE'
        },
        editCategoryContainer: {
          padding: '12px',
          backgroundColor: isDarkTheme ? '#2D3748' : '#EBF8FF',
          borderRadius: '4px',
          marginTop: '8px'
        },
        editCategoryLabel: {
          fontSize: '13px',
          marginBottom: '8px',
          color: isDarkTheme ? '#E2E8F0' : '#4A5568',
          fontWeight: '500'
        },
        manageCategoriesContainer: {
          marginTop: '8px',
          maxHeight: '150px',
          overflowY: 'auto',
          padding: '4px',
          border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
          borderRadius: '4px',
          backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',
          animation: 'fadeIn 0.2s ease'
        },
        manageCategoriesTitle: {
          fontSize: '13px',
          marginBottom: '8px',
          padding: '4px',
          borderBottom: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
          fontWeight: '500',
          color: isDarkTheme ? '#E2E8F0' : '#4A5568'
        }
      };

      return (
        <div style={categoryStyles.categoryContainer}>
          <label style={styles.label} htmlFor="category">Category</label>
          
          {/* Editing existing category */}
          {editingCategory ? (
            <div style={categoryStyles.editCategoryContainer}>
              <div style={categoryStyles.editCategoryLabel}>
                Editing category: {editingCategory}
              </div>
              <input
                type="text"
                value={editCategoryName}
                onChange={(e) => setEditCategoryName(e.target.value)}
                style={categoryStyles.newCategoryInput}
                placeholder="New category name"
              />
              <div style={categoryStyles.newCategoryActions}>
                <button
                  type="button"
                  style={{ ...categoryStyles.categoryButton, ...categoryStyles.createButton }}
                  onClick={handleEditCategory}
                  disabled={!(nameChanged || colorChanged)}
                  >
                    Save
                  </button>
                <button
                  type="button"
                  style={categoryStyles.categoryButton}
                  onClick={() => {
                    setEditingCategory(null);
                    setEditCategoryName('');
                  }}
                >
                  Cancel
                </button>
              </div>
            </div>
          ) : (
            <>
              <div style={{display: 'flex', justifyContent: 'space-between', alignItems: 'center'}}>
                {/* Standard category dropdown with "Create New" option */}
                <select
                  id="category"
                  name="category"
                  value={isNewCategory ? "__new__" : formData.category}
                  onChange={(e) => {
                    if (e.target.value === "__new__") {
                      setIsNewCategory(true);
                    } else {
                      setIsNewCategory(false);
                      setFormData(prev => ({ ...prev, category: e.target.value }));
                    }
                  }}
                  style={{ ...styles.input, height: '32px', width: 'calc(100% - 110px)' }}
                >
                  {categories.map(category => (
                    <option key={category} value={category}>
                      {category}
                    </option>
                  ))}
                  <option value="__new__">+ Create New Category</option>
                </select>
                
                {/* New Manage Categories toggle button - only if we have categories */}
                {categories.length > 1 && !isNewCategory && !editingCategory && (
                  <button
                    type="button"
                    style={{
                      ...categoryStyles.manageButton,
                      backgroundColor: showCategoryManagement ? 
                        (isDarkTheme ? '#4C51BF' : '#4299E1') : 
                        (isDarkTheme ? '#2D3748' : '#E2E8F0'),
                      color: showCategoryManagement ? 'white' : (isDarkTheme ? '#A0AEC0' : '#4A5568')
                    }}
                    onClick={() => setShowCategoryManagement(!showCategoryManagement)}
                  >
                    {showCategoryManagement ? '✕ Hide' : '⚙️ Manage'}
                  </button>
                )}
              </div>

            {/* Editing existing category UI */}
            {editingCategory && (
            <div style={{
                padding: '12px',
                backgroundColor: isDarkTheme ? '#2D3748' : '#EBF8FF',
                borderRadius: '4px',
                marginTop: '10px',
                marginBottom: '10px'
            }}>
                <div style={{
                fontSize: '13px',
                marginBottom: '8px',
                color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                fontWeight: '500'
                }}>
                Editing category: {editingCategory}
                </div>
                <input
                type="text"
                value={editCategoryName}
                onChange={(e) => setEditCategoryName(e.target.value)}
                style={modalStyles.input}
                placeholder="New category name"
                />
                <div style={{display: 'flex', gap: '8px', marginTop: '8px'}}>
                <button
                    type="button"
                    style={{
                    padding: '6px 12px',
                    backgroundColor: isDarkTheme ? '#4C51BF' : '#4299E1',
                    color: 'white',
                    border: 'none',
                    borderRadius: '4px',
                    cursor: 'pointer',
                    fontSize: '13px',
                    fontWeight: '500'
                    }}
                    onClick={handleEditCategory}
                    disabled={!(nameChanged || colorChanged)}
                  >
                    Save
                  </button>
                <button
                    type="button"
                    style={{
                    padding: '6px 12px',
                    backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
                    color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                    border: 'none',
                    borderRadius: '4px',
                    cursor: 'pointer',
                    fontSize: '13px'
                    }}
                    onClick={() => {
                    setEditingCategory(null);
                    setEditCategoryName('');
                    }}
                >
                    Cancel
                </button>
                </div>
            </div>
            )}
              
              {/* Input field for new category */}
              {isNewCategory && (
                <>
                  <input
                    type="text"
                    value={newCategoryName}
                    onChange={(e) => setNewCategoryName(e.target.value)}
                    style={categoryStyles.newCategoryInput}
                    placeholder="Enter new category name"
                    autoFocus
                  />
                  <div style={categoryStyles.newCategoryActions}>
                    <button
                      type="button"
                      style={{ ...categoryStyles.categoryButton, ...categoryStyles.createButton }}
                      onClick={handleCreateCategory}
                      disabled={!newCategoryName.trim()}
                    >
                      Create
                    </button>
                    <button
                      type="button"
                      style={categoryStyles.categoryButton}
                      onClick={() => {
                        setIsNewCategory(false);
                        setNewCategoryName('');
                      }}
                    >
                      Cancel
                    </button>
                  </div>
                </>
              )}
            </>
          )}
          
          {/* Category Management - now only shown when toggle button is clicked */}
          {categories.length > 1 && !isNewCategory && !editingCategory && showCategoryManagement && (
            <div style={categoryStyles.manageCategoriesContainer}>
              <div style={categoryStyles.manageCategoriesTitle}>
                Manage Categories
              </div>
              {categories.map(category => (
                <div 
                  key={`manage-${category}`}
                  style={{
                    ...categoryStyles.optionWithActions,
                    backgroundColor: hoveredCategory === category ? 
                      (isDarkTheme ? '#2D3748' : '#F7FAFC') : 'transparent'
                  }}
                  onMouseEnter={() => setHoveredCategory(category)}
                  onMouseLeave={() => setHoveredCategory(null)}
                >
                  {/* Color circle */}
                  <div
                    style={{
                      width: '16px',
                      height: '16px',
                      borderRadius: '50%',
                      backgroundColor: getCategoryColor(category),
                      marginRight: '8px',
                      cursor: 'pointer',
                      border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`
                    }}
                    onClick={e => {
                      e.stopPropagation();
                      setEditingCategoryColor(category);
                      setSelectedColor(categoryColors[category] || getCategoryColor(category));
                    }}
                  />

                  {/* Category name */}
                  <span style={{
                    color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                    fontWeight: category === 'General' ? '600' : '400'
                  }}>
                    {category}
                  </span>
                  <div style={{
                    ...categoryStyles.categoryActions,
                    opacity: hoveredCategory === category ? 1 : 0.3
                  }}>
                    <button
                      type="button"
                      style={{
                        ...categoryStyles.categoryActionButton,
                        ...(hoveredCategory === category ? categoryStyles.categoryActionButtonHover : {})
                      }}
                      onClick={() => {
                        setEditingCategory(category);
                        setEditCategoryName(category);
                        setShowCategoryManagement(false); // Hide management when editing
                      }}
                      title="Edit category"
                      disabled={category === 'General'}
                    >
                      {category === 'General' ? '' : '✏️ Edit'}
                    </button>
                    <button
                      type="button"
                      style={{
                        ...categoryStyles.categoryActionButton,
                        ...(hoveredCategory === category ? {
                          backgroundColor: isDarkTheme ? '#742A2A' : '#FED7D7',
                          color: isDarkTheme ? '#FEB2B2' : '#9B2C2C'
                        } : {})
                      }}
                      onClick={() => handleDeleteCategory(category)}
                      title="Delete category"
                      disabled={category === 'General'}
                    >
                      {category === 'General' ? '' : '🗑️ Delete'}
                    </button>
                  </div>
                </div>
              ))}

              {/* Color-picker inline editor */}
              {editingCategoryColor && (
                <div style={{
                  position: 'absolute',
                  bottom: '100%',    // sit right above the container
                  left: 0,           // align to the left edge (or tweak with 'left: 16px' if you prefer)
                  display: 'flex',
                  alignItems: 'center',
                  gap: '8px',
                  padding: '6px',
                  backgroundColor: isDarkTheme ? '#2D3748' : '#FFF',
                  border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
                  borderRadius: '4px',
                  boxShadow: '0 2px 6px rgba(0,0,0,0.2)',
                  zIndex: 100
                }}>
                  <input
                    type="color"
                    value={selectedColor}
                    onChange={handleColorChange}
                    style={{ width: '32px', height: '32px', padding: 0, border: 'none' }}
                  />
                  <button
                    type="button"
                    style={{ ...categoryStyles.categoryButton, ...categoryStyles.createButton }}
                    onClick={saveColorChange}
                    disabled={!selectedColor}
                  >
                    Save Color
                  </button>
                  <button
                    type="button"
                    style={categoryStyles.categoryButton}
                    onClick={() => setEditingCategoryColor(null)}
                  >
                    Cancel
                  </button>
                </div>
              )}
            </div>
          )}
        </div>
      );
    };

    // ─── Drag Handlers ───
    const handleDragStart = (e, index) => {
        setDraggedIndex(index);
        e.dataTransfer.effectAllowed = 'move';
        e.dataTransfer.setData('text/plain', index);

        // create a tiny ghost for Firefox
        const ghost = document.createElement('div');
        ghost.style.width  = '1px';
        ghost.style.height = '1px';
        document.body.appendChild(ghost);
        e.dataTransfer.setDragImage(ghost, 0, 0);
        setTimeout(() => document.body.removeChild(ghost), 0);

        // visual feedback
        e.currentTarget.style.opacity   = '0.6';
        e.currentTarget.style.transform = 'scale(1.05)';
    };

    const handleDragOver = (e, index) => {
        e.preventDefault();
        e.dataTransfer.dropEffect = 'move';
        setDragOverIndex(index);
    };

    const handleDrop = (e, index) => {
        e.preventDefault();
        if (draggedIndex === null) return;

        // clone & reorder
        const newOrdered = [...orderedFavorites];
        const [moved]    = newOrdered.splice(draggedIndex, 1);
        newOrdered.splice(index, 0, moved);

        setOrderedFavorites(newOrdered);
        setDraggedIndex(null);
        setDragOverIndex(null);

        // persist
        saveFavoriteOrder(newOrdered.map(p => p.$path));
    };

    const handleDragEnd = (e) => {
        // reset visuals & state
        e.currentTarget.style.opacity   = '1';
        e.currentTarget.style.transform = 'none';
        setDraggedIndex(null);
        setDragOverIndex(null);
    };
    
    // Query all prompt notes
    const promptNotesQuery = dc.useQuery(`
      @page 
      and path("Notes/Prompts")
      and #Prompts
    `);

    const getCategoryColor = (category) => {
        // Default fallback
        if (!category) return isDarkTheme ? '#4A5568' : '#E2E8F0';
        
        // Check if we have a saved color for this category
        if (categoryColors[category]) {
            return categoryColors[category];
        }
        
        // Generate a consistent color based on the category name
        let hash = 0;
        for (let i = 0; i < category.length; i++) {
            hash = category.charCodeAt(i) + ((hash << 5) - hash);
        }
        
        // List of colors for dark and light themes
        const colors = isDarkTheme ? 
            ['#3182CE', '#38A169', '#805AD5', '#D69E2E', '#DD6B20', '#E53E3E', '#2C7A7B', '#D53F8C'] :
            ['#63B3ED', '#68D391', '#B794F4', '#F6E05E', '#F6AD55', '#FC8181', '#4FD1C5', '#F687B3'];
        
        // Get a consistent color from the array based on hash
        return colors[Math.abs(hash) % colors.length];
        };

        // ===== STEP 4: Add color management functionality =====
        // Add these functions:
        const handleColorChange = (e) => {
        setSelectedColor(e.target.value);
        };

        const saveColorChange = () => {
        if (editingCategoryColor && selectedColor) {
            setCategoryColors(prev => ({
            ...prev,
            [editingCategoryColor]: selectedColor
            }));
            setEditingCategoryColor(null);
            setSelectedColor('');
        }
        };

    const formatTags = dc.useCallback((tags) => {
      if (!tags) return [];
      
      // Handle array of tags
      if (Array.isArray(tags)) {
        return tags
          .map(tag => tag.trim())
          .filter(tag => tag.length > 0 && tag !== "Prompts");
      }
      
      // Handle string of comma-separated tags
      if (typeof tags === 'string') {
        return tags
          .split(',')
          .map(tag => tag.trim())
          .filter(tag => tag.length > 0 && tag !== "Prompts");
      }
      
      // Handle unexpected formats by returning empty array
      return [];
    }, []);
    
    // Process the query results to extract metadata from frontmatter
    const processedPrompts = dc.useMemo(() => {
      if (!promptNotesQuery || !promptNotesQuery.length) return [];
      
      return promptNotesQuery.map(note => {
        // Extract metadata using value() method like in the Notebook component
        const processedNote = {
          ...note,
          title: note.value('title') || note.$name || 'Untitled Prompt',
          description: note.value('description') || '',
          tags: note.value('tags') || [],
          prompt: note.value('prompt') || '',
          favorite: note.value('favorite') === true, // Explicit boolean comparison
          created: note.value('created') || note.$ctime || new Date().toISOString(),
          modified: note.value('modified') || note.$mtime || new Date().toISOString(),
          $path: note.$path,
          category: note.value('category') || 'General', // Default to 'General' if no category
          lastUsed: note.value('lastUsed') || null
        };
        
        // If the prompt is stored as a JSON string, parse it
        if (typeof processedNote.prompt === 'string' && 
            (processedNote.prompt.startsWith('"') || processedNote.prompt.startsWith('{'))) {
          try {
            processedNote.prompt = JSON.parse(processedNote.prompt);
          } catch (e) {
            console.warn('Could not parse prompt as JSON for:', processedNote.title);
          }
        }
        
        // If still no prompt and we have content, try to extract it
        if ((!processedNote.prompt || processedNote.prompt === '') && note.$content) {
          const promptMatch = note.$content.match(/```\s*\n([\s\S]*?)\n\s*```/);
          if (promptMatch && promptMatch[1]) {
            processedNote.prompt = promptMatch[1];
          }
        }
        
        return processedNote;
      });
    }, [promptNotesQuery]);
    
    // Get all unique tags from all prompts
    const allTags = dc.useMemo(() => {
      const tagSet = new Set();
      
      processedPrompts.forEach(prompt => {
        const promptTags = formatTags(prompt.tags);
        promptTags.forEach(tag => tagSet.add(tag));
      });
      
      return Array.from(tagSet).sort();
    }, [processedPrompts, formatTags]);
    
    // Toggle tag selection for filtering
    const toggleTagFilter = (tag) => {
      setSelectedTags(prev => {
        if (prev.includes(tag)) {
          return prev.filter(t => t !== tag);
        } else {
          return [...prev, tag];
        }
      });
      setPage(1); 
    };
    // Handle tag input changes for autocomplete
    const handleTagInputChange = (e) => {
      const value = e.target.value;
      setTagInput(value);
      
      // Generate suggestions from all tags
      if (value.trim()) {
        const matchedTags = allTags.filter(tag => 
          tag.toLowerCase().includes(value.toLowerCase())
        );
        setTagSuggestions(matchedTags.slice(0, 5)); // Limit to 5 suggestions
        setShowTagSuggestions(matchedTags.length > 0);
      } else {
        setTagSuggestions([]);
        setShowTagSuggestions(false);
      }
    };
    
    // Add tag to form data
    const addTag = (tag) => {
      const currentTags = formData.tags ? formData.tags.split(',').map(t => t.trim()).filter(t => t) : [];
      
      // Only add if not already present
      if (!currentTags.includes(tag)) {
        const newTags = [...currentTags, tag].join(', ');
        setFormData(prev => ({ ...prev, tags: newTags }));
      }
      
      // Reset tag input
      setTagInput('');
      setShowTagSuggestions(false);
    };

    // ---------- INLINE ICON DEFINITIONS ----------
    const SvgClock = props => (
      <svg
        {...props}
        width="18" height="18"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <circle cx="12" cy="12" r="10" />
        <polyline points="12 6 12 12 16 14" />
      </svg>
    );

    const SvgEdit = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <path d="M11 4H4a2 2 0 0 0-2 2v14a2 2 0 0 0 2 2h14a2 2 0 0 0 2-2v-7" />
        <path d="M18.5 2.5a2.121 2.121 0 0 1 3 3L12 15l-4 1 1-4 9.5-9.5z" />
      </svg>
    );

    const SvgDelete = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <path d="M3 6h18" />
        <path d="M19 6v14a2 2 0 0 1-2 2H7a2 2 0 0 1-2-2V6m3 0V4a2 2 0 0 1 2-2h4a2 2 0 0 1 2 2v2" />
        <line x1="10" y1="11" x2="10" y2="17" />
        <line x1="14" y1="11" x2="14" y2="17" />
      </svg>
    );

    const SvgSearch = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <circle cx="11" cy="11" r="8" />
        <line x1="21" y1="21" x2="16.65" y2="16.65" />
      </svg>
    );

    const SvgHourglass = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <path d="M6 2h12M6 22h12M6 2L6 8c0 4 12 4 12 0V2M6 22L6 16c0-4 12-4 12 0v6" />
      </svg>
    );

    const SvgClipboard = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <path d="M16 4h2a2 2 0 0 1 2 2v14a2 2 0 0 1-2 2H6a2 2 0 0 1-2-2V6a2 2 0 0 1 2-2h2" />
        <rect x="8" y="2" width="8" height="4" rx="1" ry="1" />
      </svg>
    );

    const SvgPlus = props => (
      <svg
        {...props}
        width="24" 
        height="24"
        viewBox="0 0 24 24"
        fill="none" 
        stroke="currentColor"
        strokeWidth="2" 
        strokeLinecap="round" 
        strokeLinejoin="round"
      >
        <line x1="12" y1="5" x2="12" y2="19" />
        <line x1="5" y1="12" x2="19" y2="12" />
      </svg>
    );

    const SvgGear = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <circle cx="12" cy="12" r="3" />
        <path d="M19.4 15a1.65 1.65 0 0 0 .33 1.82l.06.06a2 2 0 0 1 0 2.83 2 2 0 0 1-2.83 0l-.06-.06a1.65 1.65 0 0 0-1.82-.33 1.65 1.65 0 0 0-1 1.51V21a2 2 0 0 1-2 2 2 2 0 0 1-2-2v-.09A1.65 1.65 0 0 0 9 19.4a1.65 1.65 0 0 0-1.82.33l-.06.06a2 2 0 0 1-2.83 0 2 2 0 0 1 0-2.83l.06-.06a1.65 1.65 0 0 0 .33-1.82 1.65 1.65 0 0 0-1.51-1H3a2 2 0 0 1-2-2 2 2 0 0 1 2-2h.09A1.65 1.65 0 0 0 4.6 9a1.65 1.65 0 0 0-.33-1.82l-.06-.06a2 2 0 0 1 0-2.83 2 2 0 0 1 2.83 0l.06.06a1.65 1.65 0 0 0 1.82.33H9a1.65 1.65 0 0 0 1-1.51V3a2 2 0 0 1 2-2 2 2 0 0 1 2 2v.09a1.65 1.65 0 0 0 1 1.51 1.65 1.65 0 0 0 1.82-.33l.06-.06a2 2 0 0 1 2.83 0 2 2 0 0 1 0 2.83l-.06.06a1.65 1.65 0 0 0-.33 1.82V9a1.65 1.65 0 0 0 1.51 1H21a2 2 0 0 1 2 2 2 2 0 0 1-2 2h-.09a1.65 1.65 0 0 0-1.51 1z" />
      </svg>
    );

    const SvgStar = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="currentColor"
        stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <polygon points="12 2 15.09 8.26 22 9.27 17 14.14 18.18 21.02 12 17.77 5.82 21.02 7 14.14 2 9.27 8.91 8.26 12 2" />
      </svg>
    );

    const SvgStarOff = props => (
      <svg
        {...props}
        width="16"
        height="16"
        viewBox="0 0 24 24"
        fill="none"
        stroke="currentColor"
        strokeWidth="2"
        strokeLinecap="round"
        strokeLinejoin="round"
      >
        <polygon points="
          12 2
          15.09 8.26
          22 9.27
          17 14.14
          18.18 21.02
          12 17.77
          5.82 21.02
          7 14.14
          2 9.27
          8.91 8.26
          12 2
        " />
      </svg>
    );

    const SvgTrash = props => (
      <svg
        {...props}
        width="16" height="16"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <polyline points="3 6 5 6 21 6" />
        <path d="M19 6l-1 14a2 2 0 0 1-2 2H8a2 2 0 0 1-2-2L5 6" />
        <path d="M10 11v6M14 11v6" />
        <path d="M9 6V4a1 1 0 0 1 1-1h4a1 1 0 0 1 1 1v2" />
      </svg>
    );

    const SvgChevronUp = props => (
      <svg
        {...props}
        width="12" height="12"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <polyline points="18 15 12 9 6 15" />
      </svg>
    );

    const SvgChevronDown = props => (
      <svg
        {...props}
        width="12" height="12"
        viewBox="0 0 24 24"
        fill="none" stroke="currentColor"
        strokeWidth="2" strokeLinecap="round" strokeLinejoin="round"
      >
        <polyline points="6 9 12 15 18 9" />
      </svg>
    );
    
    const saveRecentSearch = (term) => {
      if (!term.trim()) return;
      
      try {
        // Add the search term to the recent searches, avoiding duplicates
        // and keeping only the most recent 5
        const updatedSearches = [
          term, 
          ...recentSearches.filter(s => s !== term)
        ].slice(0, 5);
        
        setRecentSearches(updatedSearches);
        localStorage.setItem('prompt-manager-recent-searches', JSON.stringify(updatedSearches));
      } catch (err) {
        console.error('Failed to save recent search:', err);
      }
    };
    
    // OPTIMIZED FILTERING IMPLEMENTATION - Single-pass filter
    const filteredPrompts = dc.useMemo(() => {
      try {
        const startTime = performance.now();
        
        // Fast path optimization: if no filters are applied, return sorted data directly
        const noSearchTerm = !debouncedSearchTerm.trim();
        const noTags = selectedTags.length === 0;
        const noFavoriteFilter = !searchFilters.onlyFavorites;
        const allCategories = searchFilters.category === 'all';
        const noDateRange = !searchFilters.dateRange.from && !searchFilters.dateRange.to;
        
        // Save to recent searches if search term exists
        if (debouncedSearchTerm.trim()) {
          saveRecentSearch(debouncedSearchTerm.trim());
        }
        
        let results;
        
        // If no filters are applied, just apply sorting
        if (noSearchTerm && noTags && noFavoriteFilter && allCategories && noDateRange) {
          results = [...processedPrompts].sort((a, b) => {
            let valueA, valueB;
            
            // Extract values based on sort column
            if (sortColumn === 'title') {
              valueA = a.title || '';
              valueB = b.title || '';
            } else if (sortColumn === 'description') {
              valueA = a.description || '';
              valueB = b.description || '';
            } else if (sortColumn === 'tags') {
              valueA = formatTags(a.tags).join(', ');
              valueB = formatTags(b.tags).join(', ');
            } else if (sortColumn === 'created') {
              valueA = new Date(a.created || 0);
              valueB = new Date(b.created || 0);
            } else {
              valueA = a[sortColumn];
              valueB = b[sortColumn];
            }
            
            // String comparison
            if (typeof valueA === 'string' && typeof valueB === 'string') {
              const result = valueA.localeCompare(valueB);
              return sortDirection === 'asc' ? result : -result;
            }
            
            // Date comparison
            if (valueA instanceof Date && valueB instanceof Date) {
              const result = valueA - valueB;
              return sortDirection === 'asc' ? result : -result;
            }
            
            // Other type comparison
            if (valueA < valueB) return sortDirection === 'asc' ? -1 : 1;
            if (valueA > valueB) return sortDirection === 'asc' ? 1 : -1;
            return 0;
          });
        } else {
          // Apply all filters in a single pass
          results = processedPrompts
            .filter(prompt => {
              // Search term filter
              let matchesSearch = true;
              if (debouncedSearchTerm.trim()) {
                const term = debouncedSearchTerm.toLowerCase();
                const contentType = searchFilters.contentType;
                
                // Helper for safe lowercase conversion
                const safe = (value) => {
                  if (typeof value === 'string') {
                    return value.toLowerCase();
                  } else if (value === null || value === undefined) {
                    return '';
                  } else {
                    return String(value).toLowerCase();
                  }
                };
                
                if (contentType === 'all') {
                  const titleMatch = safe(prompt.title).includes(term);
                  const descMatch = safe(prompt.description).includes(term);
                  const promptTextMatch = typeof prompt.prompt === 'string' && 
                                        safe(prompt.prompt).includes(term);
                  const tagMatch = formatTags(prompt.tags || [])
                                  .some(tag => tag.toLowerCase().includes(term));
                  matchesSearch = titleMatch || descMatch || promptTextMatch || tagMatch;
                } 
                else if (contentType === 'title') {
                  matchesSearch = safe(prompt.title).includes(term);
                }
                else if (contentType === 'description') {
                  matchesSearch = safe(prompt.description).includes(term);
                }
                else if (contentType === 'prompt') {
                  matchesSearch = typeof prompt.prompt === 'string' && safe(prompt.prompt).includes(term);
                }
                else if (contentType === 'tags') {
                  matchesSearch = formatTags(prompt.tags || [])
                        .some(tag => tag.toLowerCase().includes(term));
                }
              }
              
              // Tags filter
              const matchesTags = selectedTags.length === 0 || 
                selectedTags.every(tag => formatTags(prompt.tags).includes(tag));
              
              // Favorites filter
              const matchesFavorite = !searchFilters.onlyFavorites || prompt.favorite === true;
              
              // Category filter
              const matchesCategory = searchFilters.category === 'all' || 
                prompt.category === searchFilters.category;
              
              // Date range filter
              let matchesDateRange = true;
              const { from, to } = searchFilters.dateRange;
              
              if (from || to) {
                const promptDate = new Date(prompt.created);
                
                // Check 'from' date if specified
                if (from) {
                  const fromDate = new Date(from);
                  if (promptDate < fromDate) {
                    matchesDateRange = false;
                  }
                }
                
                // Check 'to' date if specified
                if (to && matchesDateRange) {
                  const toDate = new Date(to);
                  // Set to end of day
                  toDate.setHours(23, 59, 59, 999);
                  if (promptDate > toDate) {
                    matchesDateRange = false;
                  }
                }
              }
              
              // Return true only if all conditions are met
              return matchesSearch && matchesTags && matchesFavorite && 
                     matchesCategory && matchesDateRange;
            })
            .sort((a, b) => {
              let valueA, valueB;
              
              // Extract values based on sort column
              if (sortColumn === 'title') {
                valueA = a.title || '';
                valueB = b.title || '';
              } else if (sortColumn === 'description') {
                valueA = a.description || '';
                valueB = b.description || '';
              } else if (sortColumn === 'tags') {
                valueA = formatTags(a.tags).join(', ');
                valueB = formatTags(b.tags).join(', ');
              } else if (sortColumn === 'created') {
                valueA = new Date(a.created || 0);
                valueB = new Date(b.created || 0);
              } else {
                valueA = a[sortColumn];
                valueB = b[sortColumn];
              }
              
              // String comparison
              if (typeof valueA === 'string' && typeof valueB === 'string') {
                const result = valueA.localeCompare(valueB);
                return sortDirection === 'asc' ? result : -result;
              }
              
              // Date comparison
              if (valueA instanceof Date && valueB instanceof Date) {
                const result = valueA - valueB;
                return sortDirection === 'asc' ? result : -result;
              }
              
              // Other type comparison
              if (valueA < valueB) return sortDirection === 'asc' ? -1 : 1;
              if (valueA > valueB) return sortDirection === 'asc' ? 1 : -1;
              return 0;
            });
        }
        
        const endTime = performance.now();
        setFilterTiming(endTime - startTime);
        
        return results;
      } catch (error) {
        console.error('Error in filter processing:', error);
        // Return unfiltered data as fallback
        return processedPrompts;
      }
    }, [
      processedPrompts, 
      debouncedSearchTerm, 
      searchFilters,
      selectedTags, 
      sortColumn, 
      sortDirection, 
      formatTags
    ]);

    // Apply category filtering to favorites for consistent filtering across tabs
    const categoryFilteredFavorites = dc.useMemo(() => {
      if (searchFilters.category === 'all') {
        return orderedFavorites;
      }
      
      return orderedFavorites.filter(prompt => 
        prompt.category === searchFilters.category
      );
    }, [orderedFavorites, searchFilters.category]);

    // Apply category filtering to recently used prompts
    const categoryFilteredRecentlyUsed = dc.useMemo(() => {
      if (searchFilters.category === 'all') {
        return recentlyUsedPrompts;
      }
      
      return recentlyUsedPrompts.filter(item => {
        // For recently used items, we need to handle possible differences in structure
        // Some might be stored with just path and basic info, others might have full prompt data
        const category = item.category || 'General';
        return category === searchFilters.category;
      });
    }, [recentlyUsedPrompts, searchFilters.category]);

    // Paginate the table data with minimal dependencies
    const paginatedPrompts = dc.useMemo(() => {
      const startIndex = (page - 1) * rowsPerPage; 
      return filteredPrompts.slice(startIndex, startIndex + rowsPerPage);
    }, [filteredPrompts, page, rowsPerPage]);

    // Calculate total pages based on filtered results
    const totalPages = dc.useMemo(() => {
      return Math.ceil(filteredPrompts.length / rowsPerPage);
    }, [filteredPrompts.length, rowsPerPage]);
    
    // Get categories
    const categories = dc.useMemo(() => {
      const categorySet = new Set(['General']);
      
      // Add categories from existing prompts
      processedPrompts.forEach(prompt => {
        if (prompt.category) {
          categorySet.add(prompt.category);
        }
      });
      
      // Add locally created categories that might not be in prompts yet
      localCategories.forEach(category => {
        categorySet.add(category);
      });
      
      return Array.from(categorySet).sort();
    }, [processedPrompts, localCategories]);

    // Update the handleCreateCategory function
    const handleCreateCategory = () => {
      if (!newCategoryName.trim()) return;
      
      const trimmedName = newCategoryName.trim();
      
      // Prevent duplicates
      if (categories.includes(trimmedName)) {
        setError(`Category "${trimmedName}" already exists`);
        setTimeout(() => setError(null), 3000);
        return;
      }
      
      // Add to local categories first
      setLocalCategories(prev => [...prev, trimmedName]);
      
      // Update form data with new category
      setFormData(prev => ({
        ...prev,
        category: trimmedName
      }));
      
      // Reset new category state
      setNewCategoryName('');
      setIsNewCategory(false);
      
      // Show success message
      setSuccess(`Category "${trimmedName}" created`);
      setTimeout(() => setSuccess(null), 3000);
    };

    const handleEditCategory = () => {
      if (!editingCategory) return;
      
      // Allow saving when we only have a color change (keep the same name)
      const trimmedName = editCategoryName.trim() || editingCategory;
      
      // Prevent duplicates
      if (categories.includes(trimmedName) && trimmedName !== editingCategory) {
        setError(`Category "${trimmedName}" already exists`);
        setTimeout(() => setError(null), 3000);
        return;
      }
      
      // Update local categories first
      setLocalCategories(prev => 
        prev.map(cat => cat === editingCategory ? trimmedName : cat)
      );
      
      // Handle color changes
      if (selectedColor) {
        setCategoryColors(prev => ({
          ...prev,
          [trimmedName]: selectedColor // Use the new category name as key
        }));
        
        // If the category name didn't change, also update using old name to ensure color changes immediately 
        if (trimmedName === editingCategory) {
          setCategoryColors(prev => ({
            ...prev,
            [editingCategory]: selectedColor
          }));
        }
      } else if (trimmedName !== editingCategory) {
        // If name changed but not color, transfer the existing color to the new name
        setCategoryColors(prev => {
          const newColors = { ...prev };
          if (prev[editingCategory]) {
            newColors[trimmedName] = prev[editingCategory];
          }
          return newColors;
        });
      }

      try {
        const promptsToUpdate = processedPrompts.filter(p => p.category === editingCategory);
    
   
      
      // For each prompt that uses this category, update its frontmatter
      promptsToUpdate.forEach(async (prompt) => {
        const file = dc.app.vault.getAbstractFileByPath(prompt.$path);
        if (!file) return;
        
        const content = await dc.app.vault.read(file);
        const updatedContent = content.replace(
          new RegExp(`category:\\s*${editingCategory.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}`, 'g'),
          `category: ${trimmedName}`
        );
        
        await dc.app.vault.modify(file, updatedContent);
      });
      
      // If the current form is using this category, update it
      if (formData.category === editingCategory) {
        setFormData(prev => ({
          ...prev,
          category: trimmedName
        }));
      }
      
      // Show success message including color change info if applicable
      if (selectedColor) {
        setSuccess(`Category "${editingCategory}" updated with new color and name "${trimmedName}"`);
      } else {
        setSuccess(`Category "${editingCategory}" renamed to "${trimmedName}"`);
      }
      setTimeout(() => setSuccess(null), 3000);
    } catch (err) {
      console.error('Error updating category:', err);
      setError('Failed to update category');
      setTimeout(() => setError(null), 3000);
    }
    
    // Reset editing state
    setEditingCategory(null);
    setEditCategoryName('');
    setSelectedColor(''); // Clear selected color state
    setEditingCategoryColor(null); // Clear editing color state
  };

    const handleDeleteCategory = (category) => {
    if (category === 'General') {
        setError('Cannot delete the General category');
        setTimeout(() => setError(null), 3000);
        return;
    }
    
    if (!confirm(`Are you sure you want to delete category "${category}"? All prompts in this category will be moved to "General".`)) {
        return;
    }
    
    try {
        // Remove from local categories
        setLocalCategories(prev => prev.filter(cat => cat !== category));
        
        // Find all prompts with this category
        const promptsToUpdate = processedPrompts.filter(p => p.category === category);
        
        // Update each prompt to use "General" category
        promptsToUpdate.forEach(async (prompt) => {
        const file = dc.app.vault.getAbstractFileByPath(prompt.$path);
        if (!file) return;
        
        const content = await dc.app.vault.read(file);
        const updatedContent = content.replace(
            new RegExp(`category:\\s*${category.replace(/[.*+?^${}()|[\]\\]/g, '\\$&')}`, 'g'),
            `category: General`
        );
        
        await dc.app.vault.modify(file, updatedContent);
        });
        
        // If the current form is using this category, update it
        if (formData.category === category) {
        setFormData(prev => ({
            ...prev,
            category: 'General'
        }));
        }
        
        // Show success message
        setSuccess(`Category "${category}" deleted, ${promptsToUpdate.length} prompt(s) moved to "General"`);
        setTimeout(() => setSuccess(null), 3000);
    } catch (err) {
        console.error('Error deleting category:', err);
        setError('Failed to delete category');
        setTimeout(() => setError(null), 3000);
    }
    };
    // Load recent searches from localStorage
    dc.useEffect(() => {
      try {
        const storedSearches = localStorage.getItem('prompt-manager-recent-searches');
        if (storedSearches) {
          setRecentSearches(JSON.parse(storedSearches));
        }
        
        // Also load recently used
        const recentlyUsedStore = localStorage.getItem('prompt-manager-recently-used');
        if (recentlyUsedStore) {
          setRecentlyUsedPrompts(JSON.parse(recentlyUsedStore));
        }
      } catch (err) {
        console.error('Failed to load stored data:', err);
      }
    }, []);
    
    // Toggle selection of an item for batch operations
    const toggleSelection = (promptPath) => {
      setSelectedPrompts(prev => {
        if (prev.includes(promptPath)) {
          return prev.filter(path => path !== promptPath);
        } else {
          return [...prev, promptPath];
        }
      });
    };
    
    // Toggle selection of all items on the current page
    const toggleSelectAll = () => {
      if (selectedPrompts.length === paginatedPrompts.length) {
        // If all are selected, deselect all
        setSelectedPrompts([]);
      } else {
        // Otherwise, select all
        setSelectedPrompts(paginatedPrompts.map(prompt => prompt.$path));
      }
    };
    
    // Copy prompt to clipboard
    const copyPromptToClipboard = async (prompt) => {
      try {
        if (!prompt) return;
        
        // Determine prompt text
        let promptText = '';
        if (typeof prompt.prompt === 'string') {
          promptText = prompt.prompt;
        } else if (prompt.prompt) {
          promptText = JSON.stringify(prompt.prompt, null, 2);
        }
        
        // Copy to clipboard
        await navigator.clipboard.writeText(promptText);
        
        // Mark as recently used
        markAsRecentlyUsedPrompt(prompt);
        
        // Show success message
        setSuccess('Prompt copied to clipboard!');
        setTimeout(() => setSuccess(null), 2000);
      } catch (err) {
        console.error('Failed to copy prompt:', err);
        setError('Failed to copy prompt to clipboard');
        setTimeout(() => setError(null), 3000);
      }
    };
    
    // Apply batch operation to selected prompts
    const applyBatchOperation = async () => {
      if (selectedPrompts.length === 0) return;
      
      try {
        setIsSubmitting(true);
        setError(null);
        
        // Different operations based on the selected action
        if (batchAction === 'favorite') {
          // Add to favorites
          for (const path of selectedPrompts) {
            const file = dc.app.vault.getAbstractFileByPath(path);
            if (!file) continue;
            
            const content = await dc.app.vault.read(file);
            const updatedContent = content.replace(
              /favorite: (true|false)/,
              `favorite: true`
            );
            
            await dc.app.vault.modify(file, updatedContent);
          }
          setSuccess(`Added ${selectedPrompts.length} prompts to favorites`);
        } else if (batchAction === 'unfavorite') {
          // Remove from favorites
          for (const path of selectedPrompts) {
            const file = dc.app.vault.getAbstractFileByPath(path);
            if (!file) continue;
            
            const content = await dc.app.vault.read(file);
            const updatedContent = content.replace(
              /favorite: (true|false)/,
              `favorite: false`
            );
            
            await dc.app.vault.modify(file, updatedContent);
          }
          setSuccess(`Removed ${selectedPrompts.length} prompts from favorites`);
        } else if (batchAction === 'addTag') {
          // Add tag to prompts
          if (!batchTagInput.trim()) {
            throw new Error('Please enter a tag to add');
          }
          
          const newTag = batchTagInput.trim();
          
          for (const path of selectedPrompts) {
            const file = dc.app.vault.getAbstractFileByPath(path);
            if (!file) continue;
            
            const content = await dc.app.vault.read(file);
            
            // Parse current tags
            const tagsMatch = content.match(/tags: ([^\n]+)/);
            if (tagsMatch) {
              const currentTags = tagsMatch[1];
              let tagArray = currentTags.split(',').map(tag => tag.trim());
              
              // Only add if tag doesn't already exist
              if (!tagArray.includes(newTag)) {
                tagArray.push(newTag);
                
                // Update tags in file
                const updatedContent = content.replace(
                  /tags: ([^\n]+)/,
                  `tags: ${tagArray.join(', ')}`
                );
                
                await dc.app.vault.modify(file, updatedContent);
              }
            }
          }
          
          setSuccess(`Added tag "${newTag}" to ${selectedPrompts.length} prompts`);
          setBatchTagInput('');
        } else if (batchAction === 'removeTag') {
          // Remove tag from prompts
          if (!batchTagInput.trim()) {
            throw new Error('Please enter a tag to remove');
          }
          
          const tagToRemove = batchTagInput.trim();
          
          for (const path of selectedPrompts) {
            const file = dc.app.vault.getAbstractFileByPath(path);
            if (!file) continue;
            
            const content = await dc.app.vault.read(file);
            
            // Parse current tags
            const tagsMatch = content.match(/tags: ([^\n]+)/);
            if (tagsMatch) {
              const currentTags = tagsMatch[1];
              let tagArray = currentTags.split(',').map(tag => tag.trim());
              
              // Remove the tag
              tagArray = tagArray.filter(tag => tag !== tagToRemove);
              
              // Update tags in file
              const updatedContent = content.replace(
                /tags: ([^\n]+)/,
                `tags: ${tagArray.join(', ')}`
              );
              
              await dc.app.vault.modify(file, updatedContent);
            }
          }
          
          setSuccess(`Removed tag "${tagToRemove}" from ${selectedPrompts.length} prompts`);
          setBatchTagInput('');
        } else if (batchAction === 'setCategory') {
          // Set category for prompts
          if (!batchTagInput.trim()) {
            throw new Error('Please enter a category');
          }
          
          const newCategory = batchTagInput.trim();
          
          for (const path of selectedPrompts) {
            const file = dc.app.vault.getAbstractFileByPath(path);
            if (!file) continue;
            
            const content = await dc.app.vault.read(file);
            
            // Check if category exists
            const categoryMatch = content.match(/category: ([^\n]+)/);
            if (categoryMatch) {
              // Update existing category
              const updatedContent = content.replace(
                /category: ([^\n]+)/,
                `category: ${newCategory}`
              );
              await dc.app.vault.modify(file, updatedContent);
            } else {
              // Add category if doesn't exist
              const updatedContent = content.replace(
                /---\n/,
                `---\ncategory: ${newCategory}\n`
              );
              await dc.app.vault.modify(file, updatedContent);
            }
          }
          
          setSuccess(`Set category "${newCategory}" for ${selectedPrompts.length} prompts`);
          setBatchTagInput('');
        } else if (batchAction === 'delete') {
          // Delete prompts
          for (const path of selectedPrompts) {
            const file = dc.app.vault.getAbstractFileByPath(path);
            if (!file) continue;
            
            await dc.app.vault.delete(file);
          }
          
          setSuccess(`Deleted ${selectedPrompts.length} prompts`);
        }
        
        // Reset selection after batch operation
        setSelectedPrompts([]);
        setShowBatchActions(false);
        setShowBatchConfirm(false);
        
        // Clear success message after 3 seconds
        setTimeout(() => {
          setSuccess(null);
        }, 3000);
      } catch (err) {
        console.error('Error in batch operation:', err);
        setError(err.message || 'Failed to perform batch operation');
      } finally {
        setIsSubmitting(false);
      }
    };

    const favoritePrompts = dc.useMemo(() => {
      return processedPrompts.filter(note => note.favorite === true);
    }, [processedPrompts]);
    
    // Apply the loaded order to the favorites
    dc.useEffect(() => {
      if (favoritePrompts.length === 0) {
        setOrderedFavorites([]);
        return;
      }
      
      const savedOrder = loadFavoriteOrder();
      if (savedOrder && Array.isArray(savedOrder) && savedOrder.length > 0) {
        // Create a map for quick lookup of prompts by path
        const promptsMap = new Map(favoritePrompts.map(p => [p.$path, p]));
        
        // Create ordered array based on saved paths, preserving any new favorites
        const ordered = [];
        
        // First add items in the saved order
        savedOrder.forEach(path => {
          if (promptsMap.has(path)) {
            ordered.push(promptsMap.get(path));
            promptsMap.delete(path);
          }
        });
        
        // Then add any remaining favorites not in the saved order
        if (promptsMap.size > 0) {
            ordered.push(...promptsMap.values());
        }
        
        setOrderedFavorites(ordered);
      } else {
        // If no saved order, use the default order
        setOrderedFavorites(favoritePrompts);
      }
    }, [favoritePrompts]);
    
    // Handle form input changes
    const handleInputChange = (e) => {
      const { name, value, type, checked } = e.target;
      setFormData(prev => ({
        ...prev,
        [name]: type === 'checkbox' ? checked : value
      }));
    };
    
     // Handle form submission
    const handleSubmit = async (e) => {
      e.preventDefault();
      
      try {
        setIsSubmitting(true);
        setError(null);
        
        // Validate form data
        if (!formData.title.trim()) {
          throw new Error('Title is required');
        }
        
        if (!formData.prompt.trim()) {
          throw new Error('Prompt text is required');
        }
        
        // Process tags - split by commas and trim
        const processedTags = formData.tags
          .split(',')
          .map(tag => tag.trim())
          .filter(tag => tag.length > 0)
          .join(', ');
        
        // Add "Prompts" tag if not already in the tags
        let tagsList = formData.tags.split(',').map(tag => tag.trim()).filter(tag => tag.length > 0);
        if (!tagsList.includes('Prompts')) {
          tagsList.push('Prompts');
        }
        
        // Format tags as YAML array for frontmatter
        const yamlTags = tagsList.map(tag => `  - ${tag}`).join('\n');
        
        // Create note content with YAML frontmatter
        const content = `---
title: ${formData.title}
description: ${formData.description}
tags:
${yamlTags}
prompt: ${JSON.stringify(formData.prompt)}
favorite: ${formData.favorite}
category: ${formData.category}
created: ${new Date().toISOString()}
modified: ${new Date().toISOString()}
---

<div style="
  max-width: 40%;
  margin: 20px auto;
  padding: 30px;
  background-color: var(--background-primary);
  border: 1px solid var(--background-modifier-border);
  border-radius: 12px;
  box-shadow: 0 4px 6px rgba(0, 0, 0, 0.1);
  font-family: var(--font-interface);
">
  <h1 style="
    margin: 0 0 16px 0;
    color: var(--text-normal);
    font-size: 28px;
    font-weight: 600;
    letter-spacing: -0.02em;
  ">${formData.title}</h1>
  
  ${formData.description ? `<h2 style="
    margin: 0 0 20px 0;
    color: var(--text-muted);
    font-size: 18px;
    font-weight: 400;
    line-height: 1.5;
    border-left: 3px solid var(--interactive-accent);
    padding-left: 12px;
  ">${formData.description}</h2>` : ''}
  
  <h3 style="
    display: inline-block;
    margin: 0 0 20px 0;
    padding: 6px 14px;
    background-color: var(--interactive-accent);
    color: var(--text-on-accent);
    font-size: 14px;
    font-weight: 500;
    border-radius: 20px;
    text-transform: uppercase;
    letter-spacing: 0.05em;
  ">${formData.category}</h3>
  
  <div style="
    margin: 24px 0;
    background-color: var(--code-background);
    border-radius: 8px;
    overflow: hidden;
  ">
    <div style="
      padding: 8px 16px;
      background-color: var(--background-modifier-border);
      font-size: 12px;
      color: var(--text-muted);
      font-weight: 500;
      display: flex;
      justify-content: space-between;
      align-items: center;
    ">
      <span>PROMPT</span>
      <button onclick="navigator.clipboard.writeText(this.closest('div').nextElementSibling.querySelector('code').textContent)" style="
        background: none;
        border: none;
        color: var(--text-muted);
        cursor: pointer;
        padding: 4px 8px;
        border-radius: 4px;
        font-size: 12px;
        transition: all 0.2s ease;
      " onmouseover="this.style.backgroundColor='var(--background-modifier-hover)'" onmouseout="this.style.backgroundColor='transparent'">
        📋 Copy
      </button>
    </div>
    <pre style="margin: 0; padding: 16px; overflow-x: auto;"><code style="
      font-family: 'SFMono-Regular', Consolas, 'Liberation Mono', Menlo, monospace;
      font-size: 14px;
      line-height: 1.6;
      color: var(--text-normal);
    ">${formData.prompt}</code></pre>
  </div>
  
  ${tagsList.length > 0 ? `<div style="
    display: flex;
    flex-wrap: wrap;
    gap: 8px;
    margin-top: 24px;
  ">
    ${tagsList.map(tag => `<span style="
      display: inline-flex;
      align-items: center;
      padding: 6px 16px;
      background-color: var(--background-secondary);
      color: var(--text-muted);
      border-radius: 20px;
      font-size: 13px;
      font-weight: 500;
      border: 1px solid var(--background-modifier-border);
      transition: all 0.2s ease;
    ">${tag}</span>`).join('')}
  </div>` : ''}
</div>

<style>
  /* Ensure the card is responsive on smaller screens */
  @media (max-width: 768px) {
    div[style*="max-width: 40%"] {
      max-width: 90% !important;
    }
  }
</style>
`;
        
        // Create the note file in the vault
        // Sanitize the title for filename
        const safeTitle = formData.title
          .replace(/[\\/:*?"<>|]/g, '-') // Replace invalid filename chars
          .replace(/\s+/g, '-')         // Replace spaces with hyphens
          .replace(/-+/g, '-');         // Collapse multiple hyphens
        
        // Ensure the Notes/Prompts directory exists
        await dc.app.vault.createFolder('Notes/Prompts').catch(err => {
          // Ignore error if folder already exists
          if (!err.message.includes('already exists')) {
            throw err;
          }
        });
        
        console.log("Creating prompt with content:", content);
        
        // Create the note file
        const filePath = `Notes/Prompts/${safeTitle}.md`;
        await dc.app.vault.create(filePath, content);
        
        // Show success message and reset form
        setSuccess(`Prompt "${formData.title}" created successfully`);
        setFormData({
          title: '',
          description: '',
          tags: '',
          prompt: '',
          favorite: false,
          category: 'General'
        });
        setShowForm(false);
        
        // Clear success message after 3 seconds
        setTimeout(() => {
          setSuccess(null);
        }, 3000);
        
      } catch (err) {
        console.error('Error creating prompt:', err);
        setError(err.message || 'Failed to create prompt');
      } finally {
        setIsSubmitting(false);
      }
    };
    
    // Handle sort changes
    const handleSort = (column) => {
      if (sortColumn === column) {
        // Toggle direction if already sorting by this column
        setSortDirection(prev => prev === 'asc' ? 'desc' : 'asc');
      } else {
        // Set new column and default to ascending
        setSortColumn(column);
        setSortDirection('asc');
      }
    };
    
    // Function to format date strings
    const formatDate = (dateStr) => {
      if (!dateStr) return 'N/A';
      
      try {
        const date = new Date(dateStr);
        return new Intl.DateTimeFormat('en-US', {
          year: 'numeric',
          month: 'short',
          day: 'numeric'
        }).format(date);
      } catch (err) {
        return dateStr;
      }
    };
    
    // Function to truncate text - with improved type handling
    const truncateText = (text, maxLength = 100) => {
      // First ensure we have a string
      const stringValue = text === null || text === undefined ? '' : String(text);
      
      // Then perform truncation
      if (stringValue.length <= maxLength) return stringValue;
      return stringValue.substring(0, maxLength) + '...';
    };
   
    
    // Open the prompt note in Obsidian
    const openPromptNote = (note) => {
      if (note && note.$path) {
        const file = dc.app.vault.getAbstractFileByPath(note.$path);
        if (file) {
          dc.app.workspace.getLeaf().openFile(file);
        }
      }
    };
    
    // Toggle favorite status for a prompt
    const toggleFavorite = async (note) => {
      if (!note || !note.$path) return;
      
      try {
        // Get the file content
        const file = dc.app.vault.getAbstractFileByPath(note.$path);
        if (!file) return;
        
        const content = await dc.app.vault.read(file);
        
        // Update the favorite status in frontmatter
        const newFavoriteStatus = !note.favorite;
        const updatedContent = content.replace(
          /favorite: (true|false)/,
          `favorite: ${newFavoriteStatus}`
        );
        
        // Write the updated content back
        await dc.app.vault.modify(file, updatedContent);
        
        // Show success message
        setSuccess(`Prompt ${newFavoriteStatus ? 'added to' : 'removed from'} favorites`);
        
        // Clear success message after 3 seconds
        setTimeout(() => {
          setSuccess(null);
        }, 3000);
      } catch (err) {
        console.error('Error updating favorite status:', err);
        setError('Failed to update favorite status');
      }
    };
    
    // Delete a prompt
    const deletePrompt = async (note) => {
      if (!note || !note.$path) return;
      
      // Confirm deletion
      if (!confirm(`Are you sure you want to delete the prompt "${note.title}"?`)) {
        return;
      }
      
      try {
        const file = dc.app.vault.getAbstractFileByPath(note.$path);
        if (!file) return;
        
        await dc.app.vault.delete(file);
        
        setSuccess(`Prompt "${note.title}" deleted successfully`);
        
        // Clear success message after 3 seconds
        setTimeout(() => {
          setSuccess(null);
        }, 3000);
      } catch (err) {
        console.error('Error deleting prompt:', err);
        setError('Failed to delete prompt');
      }
    };
    
    // Initial load of categories from localStorage
    dc.useEffect(() => {
      try {
        const storedCategories = localStorage.getItem('prompt-manager-categories');
        if (storedCategories) {
          const parsedCategories = JSON.parse(storedCategories);
          // Ensure 'General' is always there
          if (!parsedCategories.includes('General')) {
            parsedCategories.unshift('General');
          }
          setLocalCategories(parsedCategories);
        }
      } catch (err) {
        console.error('Failed to load categories from localStorage:', err);
      }
    }, []);

    // Save categories to localStorage when they change
    dc.useEffect(() => {
      try {
        localStorage.setItem('prompt-manager-categories', JSON.stringify(categories));
      } catch (err) {
        console.error('Failed to save categories to localStorage:', err);
      }
    }, [categories]);

    const nextPage = () => {
      if (page < totalPages) {
        setPage(page + 1);
      }
    };

    const prevPage = () => {
      if (page > 1) {
        setPage(page - 1);
      }
    };

    // Handle page changes
    const handlePageChange = (newPage) => {
      if (newPage >= 1 && newPage <= totalPages) {
        setPage(newPage);
      }
    };
    
    // Render the batch action bar
    const renderBatchActionBar = () => {
      if (!showBatchActions || selectedPrompts.length === 0) return null;
      
      return (
        <div style={styles.batchActionBar}>
          <div>
            <span style={{ fontWeight: '500' }}>{selectedPrompts.length} prompts selected</span>
          </div>
          <div style={styles.batchActionGroup}>
            {/* Batch action selector */}
            <select
              style={styles.filterSelect}
              value={batchAction}
              onChange={(e) => setBatchAction(e.target.value)}
            >
              <option value="">Select action...</option>
              <option value="favorite">Add to favorites</option>
              <option value="unfavorite">Remove from favorites</option>
              <option value="addTag">Add tag</option>
              <option value="removeTag">Remove tag</option>
              <option value="setCategory">Set category</option>
              <option value="delete">Delete prompts</option>
            </select>
            
            {/* Tag input for batch tag operations */}
            {['addTag', 'removeTag', 'setCategory'].includes(batchAction) && (
              <input
                type="text"
                style={styles.batchTagInput}
                value={batchTagInput}
                onChange={(e) => setBatchTagInput(e.target.value)}
                placeholder={
                  batchAction === 'addTag' ? 'Tag to add...' : 
                  batchAction === 'removeTag' ? 'Tag to remove...' :
                  'Category name...'
                }
              />
            )}
            
            {/* Apply button */}
            <button
              style={styles.batchActionButton}
              onClick={() => {
                if (batchAction === 'delete') {
                  setShowBatchConfirm(true);
                } else {
                  applyBatchOperation();
                }
              }}
              disabled={!batchAction || 
                (['addTag', 'removeTag', 'setCategory'].includes(batchAction) && !batchTagInput.trim())}
            >
              Apply
            </button>
            
            {/* Cancel button */}
            <button
              style={styles.batchActionButton}
              onClick={() => {
                setShowBatchActions(false);
                setSelectedPrompts([]);
                setBatchAction('');
                setBatchTagInput('');
              }}
            >
              Cancel
            </button>
          </div>
        </div>
      );
    };
    
    // Render batch confirmation dialog
    const renderBatchConfirmDialog = () => {
      if (!showBatchConfirm) return null;
      
      return (
        <div style={styles.batchConfirmOverlay}>
          <div style={styles.batchConfirmDialog}>
            <h3 style={styles.batchConfirmTitle}>Confirm Delete</h3>
            <p style={styles.batchConfirmText}>
              Are you sure you want to delete {selectedPrompts.length} prompts? This action cannot be undone.
            </p>
            <div style={styles.batchConfirmButtons}>
              <button
                style={styles.batchCancelButton}
                onClick={() => setShowBatchConfirm(false)}
              >
                Cancel
              </button>
              <button
                style={styles.batchConfirmButton}
                onClick={() => {
                  applyBatchOperation();
                  setShowBatchConfirm(false);
                }}
              >
                Delete
              </button>
            </div>
          </div>
        </div>
      );
    };

    const tabStyles = {
      tabsContainer: {
        display: 'flex',
        borderBottom: `2px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
        marginBottom: '24px',
        position: 'relative'
      },
      tabButton: {
        padding: '12px 20px',
        backgroundColor: 'transparent',
        border: 'none',
        borderBottom: '3px solid transparent',
        fontSize: '15px',
        fontWeight: '500',
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        cursor: 'pointer',
        transition: 'all 0.2s ease',
        position: 'relative',
        bottom: '-2px',
        marginRight: '8px'
      },
      activeTabButton: {
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        borderBottomColor: isDarkTheme ? '#4C51BF' : '#4299E1',
        fontWeight: '600'
      },
      tabContent: {
        padding: '10px 0'
      },
      tabIcon: {
        marginRight: '8px',
        verticalAlign: 'middle',
        fontSize: '16px'
      }
    };
    
    // Render recently used section
    const renderRecentlyUsedSection = () => {
    if (categoryFilteredRecentlyUsed.length === 0) {
        return (
        <div style={styles.emptyState}>
            <p style={styles.emptyStateText}>
            {searchFilters.category !== 'all' ? 
                `No recently used prompts in the "${searchFilters.category}" category` : 
                'No recently used prompts found'}
            </p>
            <button
            style={styles.addButton}
            onClick={() => setShowForm(true)}
            >
            + New Prompt
            </button>
        </div>
        );
    }
    
    return (
        <div style={styles.recentlyUsedSection}>
        <div style={styles.favoritesScrollContainer}>
            {/* Left arrow */}
            <div
            style={styles.scrollIndicatorLeft}
            id="recently-used-scroll-left"
            onClick={() => {
                const c = document.getElementById('recently-used-scroll-container');
                if (c) c.scrollBy({ left: -300, behavior: 'smooth' });
            }}
            >◀</div>

            {/* Two-row grid with the same structure as favorites */}
            <div
            id="recently-used-scroll-container"
            style={{
                ...styles.favoritesScroll,
                flexWrap: 'wrap',
                justifyContent: 'flex-start',
                gap: '24px',
                paddingLeft: '20px',
                paddingRight: '20px'
            }}
            >
            {categoryFilteredRecentlyUsed.map((item, index) => {
                const promptText = typeof item.prompt === 'string'
                ? item.prompt
                : JSON.stringify(item.prompt);
                const promptPreview = promptText
                .split('\n')
                .slice(0, 3)
                .join('\n');
                
                // Try to find the full prompt as a fallback, not the primary source
                const fullPrompt = processedPrompts.find(p => 
                p.$path === item.$path || p.$path === item.path
                );
                
                // Use the item's data directly, fall back to fullPrompt only if needed
                const title = item.title || (fullPrompt ? fullPrompt.title : 'Untitled');
                const description = item.description || (fullPrompt ? fullPrompt.description : '');
                const tags = item.tags || (fullPrompt ? fullPrompt.tags : []);
                const category = item.category || (fullPrompt ? fullPrompt.category : 'General');
                const isFavorite = item.favorite !== undefined ? item.favorite : 
                                (fullPrompt ? fullPrompt.favorite : false);

                return (
                <div
                    key={`recent-${index}-${title}`}
                    style={styles.card}
                    onClick={() => openPromptNote({ $path: item.$path || item.path })}
                >
                    <div style={styles.cardContentWrapper}>
                    <div style={styles.cardTitleSection}>
                        <div style={styles.cardTitle}>
                        {title}
                        </div>
                    </div>

                    <div style={styles.cardDescriptionSection}>
                        {description ? (
                        <div style={styles.cardDescription}>
                            {description}
                        </div>
                        ) : (
                        <div style={{ ...styles.cardDescription, fontStyle: 'italic', opacity: 0.7 }}>
                            No description
                        </div>
                        )}
                    </div>

                    <div style={styles.cardPromptPreviewSection}>
                        <div style={styles.cardPromptPreview}>
                        {promptPreview.length > 0
                            ? truncateText(promptPreview, 150)
                            : <span style={{ fontStyle: 'italic', opacity: 0.7 }}>No preview available</span>}
                        </div>
                    </div>

                    <div style={styles.cardTagsSection}>
                        <div style={styles.cardTags}>
                        {formatTags(tags).map((tag, i) => (
                            <span
                            key={`recent-tag-${i}`}
                            style={styles.tag}
                            onClick={e => {
                                e.stopPropagation();
                                toggleTagFilter(tag);
                            }}
                            >
                            {tag}
                            </span>
                        ))}
                        {formatTags(tags).length === 0 && (
                            <span style={{ 
                            fontStyle: 'italic', 
                            opacity: 0.7, 
                            color: isDarkTheme ? '#A0AEC0' : '#94A3B8'
                            }}>
                            No tags
                            </span>
                        )}
                        </div>
                    </div>
                    <div style={styles.cardActionsSection}>
                        <div style={styles.cardActions}>
                        <button
                            style={styles.cardButton}
                            onClick={e => {
                                e.stopPropagation();
                                const textToCopy = item.prompt || (fullPrompt ? fullPrompt.prompt : '');
                                navigator.clipboard.writeText(textToCopy);
                                // Show success message
                                setSuccess('Copied to clipboard!');
                                setTimeout(() => setSuccess(null), 2000);
                                // Also mark as recently used if we have the full prompt
                                if (fullPrompt) {
                                markAsRecentlyUsedPrompt(fullPrompt);
                                }
                            }}
                            title="Copy prompt"
                            >
                            <SvgClipboard />
                            </button>
                        {fullPrompt && (
                            <button
                            style={styles.cardButton}
                            onClick={e => {
                                e.stopPropagation();
                                toggleFavorite(fullPrompt);
                            }}
                            title={isFavorite ? "Remove from favorites" : "Add to favorites"}
                            >
                            {isFavorite ? <SvgStar /> : <SvgStarOff />}
                            </button>
                        )}
                        </div>
                    </div>
                    
                    {/* Category badge */}
                    <div 
                        style={{
                        ...styles.categoryBadge,
                        backgroundColor: getCategoryColor(fullPrompt ? fullPrompt.category : item.category),
                        color: isDarkTheme 
                            ? 'rgba(255, 255, 255, 0.9)' 
                            : 'rgba(0, 0, 0, 0.8)'
                        }}
                        title={`Category: ${fullPrompt ? (fullPrompt.category || 'General') : (item.category || 'General')}`}
                    >
                        {fullPrompt ? (fullPrompt.category || 'General') : (item.category || 'General')}
                    </div>
                    </div>
                </div>
                );
            })}
            </div>

            {/* Right arrow */}
            <div
            style={styles.scrollIndicatorRight}
            id="recently-used-scroll-right"
            onClick={() => {
                const c = document.getElementById('recently-used-scroll-container');
                if (c) c.scrollBy({ left: 300, behavior: 'smooth' });
            }}
            >▶</div>
        </div>
        </div>
    );
    };
    
    // Render table header with checkbox
    const renderTableHeader = () => (
      <thead style={styles.tableHead}>
        <tr>
          <th style={{ ...styles.tableCell, width: '20%' }}>Title</th>
          <th style={{ ...styles.tableCell, width: '30%' }}>Description</th>
          <th style={{ ...styles.tableCell, width: '20%' }}>Category</th>
          <th style={{ ...styles.tableCell, width: '10%' }}>Tags</th>
          <th style={{ ...styles.tableCell, width: '10%' }}>Created</th>
          <th style={{ ...styles.tableCell, width: '5%', textAlign: 'center' }}>Actions</th>
        </tr>
      </thead>
    );

    // Render tag autocomplete for the form
    const renderTagAutocomplete = () => (
      <div style={styles.formGroup}>
        <label style={styles.label} htmlFor="tags">Tags</label>
        <input
          id="tags"
          name="tags"
          type="text"
          value={tagInput}
          onChange={handleTagInputChange}
          onFocus={() => setShowTagSuggestions(true)}
          onBlur={() => setTimeout(() => setShowTagSuggestions(false), 100)}
          style={styles.input}
          placeholder="Type to search or add tags"
        />
        {showTagSuggestions && (
          <ul style={styles.tagSuggestionContainer}>
            {tagSuggestions.map(tag => (
              <li
                key={tag}
                style={styles.tagSuggestionItem}
                onMouseDown={() => addTag(tag)}
              >
                {tag}
              </li>
            ))}
          </ul>
        )}
      </div>
    );

    const modalStyles = {
      overlay: {
        position: 'fixed',
        top: 0,
        left: 0,
        right: 0,
        bottom: 0,
        backgroundColor: 'rgba(0, 0, 0, 0.5)',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        zIndex: 1000,
        backdropFilter: 'blur(3px)',
        transition: 'all 0.3s ease'
      },
      modal: {
        backgroundColor: isDarkTheme ? '#1E2A3B' : 'white',
        borderRadius: '12px',
        padding: '30px',
        width: '500px',
        maxWidth: '90%',
        maxHeight: '90vh',
        overflowY: 'auto',
        boxShadow: '0 10px 25px rgba(0, 0, 0, 0.3)',
        animation: 'modalFadeIn 0.3s ease',
        position: 'relative'
      },
      modalHeader: {
        display: 'flex',
        justifyContent: 'space-between',
        alignItems: 'center',
        marginBottom: '24px'
      },
      modalTitle: {
        fontSize: '24px',
        fontWeight: '600',
        color: isDarkTheme ? '#E2E8F0' : '#2D3748',
        margin: 0
      },
      closeButton: {
        background: 'none',
        border: 'none',
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        fontSize: '24px',
        cursor: 'pointer',
        padding: '5px 10px',
        borderRadius: '4px',
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#2D3748' : '#EDF2F7',
          color: isDarkTheme ? '#E2E8F0' : '#4A5568'
        }
      },
      formGroup: {
        marginBottom: '20px',
        width: '100%',
        maxWidth: '440px'
      },
      inputWrapper: {
        position: 'relative',
        width: '100%',
        maxWidth: '440px'
      },
      label: {
        display: 'block',
        marginBottom: '8px',
        fontWeight: '500',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        fontSize: '14px'
      },
      input: {
        width: '100%',
        padding: '10px 12px',
        borderRadius: '6px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#2D3748' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        fontSize: '14px',
        transition: 'all 0.2s ease',
        '&:focus': {
          borderColor: isDarkTheme ? '#4C51BF' : '#4299E1',
          outline: 'none',
          boxShadow: `0 0 0 3px ${isDarkTheme ? 'rgba(76, 81, 191, 0.3)' : 'rgba(66, 153, 225, 0.3)'}`
        }
      },
      textarea: {
        width: '100%',
        padding: '10px 12px',
        borderRadius: '6px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#2D3748' : 'white',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        minHeight: '120px',
        fontFamily: 'monospace',
        fontSize: '14px',
        transition: 'all 0.2s ease',
        '&:focus': {
          borderColor: isDarkTheme ? '#4C51BF' : '#4299E1',
          outline: 'none',
          boxShadow: `0 0 0 3px ${isDarkTheme ? 'rgba(76, 81, 191, 0.3)' : 'rgba(66, 153, 225, 0.3)'}`
        }
      },
      select: {
        width: '100%',
        padding: '10px 12px',
        borderRadius: '6px',
        border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
        backgroundColor: isDarkTheme ? '#2D3748' : '#f7fafc',
        color: isDarkTheme ? '#E2E8F0' : 'inherit',
        fontSize: '14px',
        appearance: 'none',
        backgroundImage: `url('data:image/svg+xml;charset=US-ASCII,${encodeURIComponent(
          `<svg width="12" height="12" viewBox="0 0 12 12" fill="none" xmlns="http://www.w3.org/2000/svg"><path d="M2.5 4.5L6 8L9.5 4.5" stroke="${
            isDarkTheme ? '#A0AEC0' : '#4A5568'
          }" stroke-width="1.5" stroke-linecap="round" stroke-linejoin="round"/></svg>`
        )}')`,
        backgroundRepeat: 'no-repeat',
        backgroundPosition: 'right 12px center',
        backgroundSize: '12px',
        height: '40px'
      },
      formActions: {
        display: 'flex',
        justifyContent: 'flex-end',
        gap: '12px',
        marginTop: '30px'
      },
      cancelButton: {
        padding: '10px 16px',
        backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568',
        border: 'none',
        borderRadius: '6px',
        cursor: 'pointer',
        fontSize: '14px',
        fontWeight: '500',
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#2D3748' : '#CBD5E0'
        }
      },
      submitButton: {
        padding: '10px 16px',
        backgroundColor: isDarkTheme ? '#4C51BF' : '#4299E1',
        color: 'white',
        border: 'none',
        borderRadius: '6px',
        cursor: 'pointer',
        fontSize: '14px',
        fontWeight: '500',
        transition: 'all 0.2s ease',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#434190' : '#3182CE'
        }
      },
      floatingActionButton: {
        position: 'fixed',
        bottom: '30px',
        right: '30px',
        width: '56px',
        height: '56px',
        borderRadius: '50%',
        backgroundColor: isDarkTheme ? '#4C51BF' : '#4299E1',
        color: 'white',
        display: 'flex',
        alignItems: 'center',
        justifyContent: 'center',
        boxShadow: '0 4px 12px rgba(0, 0, 0, 0.2)',
        cursor: 'pointer',
        border: 'none',
        zIndex: 1000,
        transition: 'all 0.2s ease',
        '&:hover': {
          transform: 'scale(1.05)',
          boxShadow: '0 6px 16px rgba(0, 0, 0, 0.3)',
          backgroundColor: isDarkTheme ? '#434190' : '#3182CE'
        },
        '&:active': {
          transform: 'scale(0.95)'
        }
      },
      tagPillsContainer: {
        display: 'flex',
        flexWrap: 'wrap',
        gap: '8px',
        marginTop: '8px',
        minHeight: '32px'
      },
      tagPill: {
        display: 'flex',
        alignItems: 'center',
        gap: '6px',
        padding: '6px 10px',
        backgroundColor: isDarkTheme ? '#2D3748' : '#EBF8FF',
        color: isDarkTheme ? '#90CDF4' : '#2C5282',
        borderRadius: '20px',
        fontSize: '13px',
        fontWeight: '500',
        transition: 'all 0.2s ease'
      },
      removeTagButton: {
        background: 'none',
        border: 'none',
        cursor: 'pointer',
        color: isDarkTheme ? '#A0AEC0' : '#4A5568',
        display: 'inline-flex',
        alignItems: 'center',
        justifyContent: 'center',
        fontSize: '14px',
        width: '16px',
        height: '16px',
        borderRadius: '50%',
        padding: 0,
        margin: 0,
        marginLeft: '4px',
        opacity: 0.7,
        transition: 'all 0.15s ease',
        lineHeight: 1,
        verticalAlign: 'middle',
        '&:hover': {
          opacity: 1,
          backgroundColor: isDarkTheme ? 'rgba(255,255,255,0.1)' : 'rgba(0,0,0,0.05)',
          color: isDarkTheme ? '#E2E8F0' : '#2D3748'
        }
      },
      checkboxContainer: {
        display: 'flex',
        alignItems: 'center',
        gap: '10px'
      },
      checkbox: {
        width: '18px',
        height: '18px',
        cursor: 'pointer'
      },
      checkboxLabel: {
        display: 'flex',
        alignItems: 'center',
        gap: '8px',
        cursor: 'pointer',
        fontSize: '14px',
        color: isDarkTheme ? '#E2E8F0' : '#4A5568'
      },
      categoryRow: {
        display: 'flex',
        alignItems: 'center',
        gap: '8px'
      },
      gearIcon: {
        color: isDarkTheme ? '#A0AEC0' : '#718096',
        cursor: 'pointer',
        transition: 'all 0.2s ease',
        padding: '6px',
        borderRadius: '4px',
        '&:hover': {
          backgroundColor: isDarkTheme ? '#2D3748' : '#EDF2F7',
          color: isDarkTheme ? '#E2E8F0' : '#4A5568',
          transform: 'rotate(30deg)'
        }
      }
    };
    
    // Main render function
    return (
      <div style={styles.container}>
        <div style={styles.header}>
          <div style={{ display: 'flex', alignItems: 'center', gap: '16px', flex: 1 }}>            
            <div style={{
              display: 'flex', 
              alignItems: 'center',
              gap: '12px',
              flex: 1
            }}>
              {/* Search input without the icon */}
              <input
                type="text"
                placeholder="Search prompts..."
                value={searchTerm}
                onChange={(e) => {
                  setSearchTerm(e.target.value);
                  setPage(1); // Reset to first page when searching
                }}
                style={{ 
                  ...styles.searchInput, 
                  paddingRight: '12px', // Reduced padding since no icon
                  minWidth: '250px', 
                  maxWidth: '400px',
                  flex: 1
                }}
              />
              
              {/* Filters button moved next to search input */}
              <button 
                style={{
                  ...styles.filterToggle,
                  backgroundColor: showFilters || selectedTags.length > 0 || searchFilters.onlyFavorites ? 
                    (isDarkTheme ? '#4A5568' : '#EBF8FF') : 'transparent'
                }}
                onClick={() => setShowFilters(!showFilters)}
                title="Advanced search options"
              >
                <span>Filters</span>
                <span>{selectedTags.length > 0 || searchFilters.onlyFavorites ? 
                  `(${selectedTags.length + (searchFilters.onlyFavorites ? 1 : 0)})` : ''}
                </span>
                <span>
                  {showFilters
                    ? <SvgChevronUp />
                    : <SvgChevronDown />}
                </span>
              </button>
            </div>
          </div>
        </div>

        {/* Category bar moved OUTSIDE the header, below search box */}
        <div style={{
          marginTop: '16px',
          marginBottom: '16px',
          width: '100%'
        }}>
          <div style={{
            display: 'flex',
            alignItems: 'center',
            marginBottom: '8px',
          }}>
            <span style={{
              fontSize: '14px',
              fontWeight: '500',
              color: isDarkTheme ? '#E2E8F0' : '#4A5568',
            }}>
            </span>
          </div>
          
          <div style={{
            display: 'flex',
            flexWrap: 'nowrap',
            gap: '8px',
            overflowX: 'auto',
            padding: '4px 0',
            scrollbarWidth: 'thin',
            msOverflowStyle: 'none', // Hide scrollbar in IE and Edge
            '&::-webkit-scrollbar': { height: '4px' },
            '&::-webkit-scrollbar-track': { backgroundColor: 'transparent' },
            '&::-webkit-scrollbar-thumb': { 
              backgroundColor: isDarkTheme ? '#4A5568' : '#CBD5E0',
              borderRadius: '4px'
            }
          }}>
            {/* "All" button */}
            <button 
              style={{
                fontSize: '13px',
                padding: '6px 16px',
                borderRadius: '20px',
                backgroundColor: searchFilters.category === 'all' 
                  ? (isDarkTheme ? '#4A5568' : '#E2E8F0')
                  : 'transparent',
                color: searchFilters.category === 'all'
                  ? (isDarkTheme ? 'white' : '#2D3748')
                  : (isDarkTheme ? '#A0AEC0' : '#718096'),
                cursor: 'pointer',
                transition: 'all 0.2s ease',
                border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
                fontWeight: searchFilters.category === 'all' ? '500' : '400',
                textTransform: 'uppercase',
                letterSpacing: '0.05em',
                fontSize: '11px',
                whiteSpace: 'nowrap',
                flexShrink: 0
              }}
              onClick={() => {
                setSearchFilters(prev => ({
                  ...prev,
                  category: 'all'
                }));
                setPage(1);
              }}
            >
              All Categories
            </button>
            
            {/* Category buttons */}
            {categories.map(category => (
              <button
                key={`category-bar-${category}`}
                style={{
                  fontSize: '11px',
                  padding: '6px 16px',
                  borderRadius: '20px',
                  backgroundColor: searchFilters.category === category 
                    ? getCategoryColor(category)
                    : (isDarkTheme ? 'rgba(45, 55, 72, 0.4)' : 'rgba(237, 242, 247, 0.7)'),
                  color: searchFilters.category === category
                    ? (isDarkTheme ? 'rgba(255, 255, 255, 0.95)' : 'rgba(0, 0, 0, 0.85)')
                    : (isDarkTheme ? '#A0AEC0' : '#4A5568'),
                  cursor: 'pointer',
                  transition: 'all 0.2s ease',
                  border: `1px solid ${searchFilters.category === category 
                    ? getCategoryColor(category)
                    : (isDarkTheme ? '#2D3748' : '#E2E8F0')}`,
                  fontWeight: searchFilters.category === category ? '600' : '500',
                  textTransform: 'uppercase',
                  letterSpacing: '0.05em',
                  boxShadow: searchFilters.category === category 
                    ? `0 2px 4px ${isDarkTheme ? 'rgba(0,0,0,0.3)' : 'rgba(0,0,0,0.1)'}` 
                    : 'none',
                  whiteSpace: 'nowrap',
                  flexShrink: 0,
                  opacity: searchFilters.category === category ? 1 : 0.85,
                  '&:hover': {
                    opacity: 1,
                    backgroundColor: searchFilters.category !== category 
                      ? (isDarkTheme ? 'rgba(45, 55, 72, 0.6)' : 'rgba(237, 242, 247, 0.9)')
                      : getCategoryColor(category)
                  }
                }}
                onClick={() => {
                  setSearchFilters(prev => ({
                    ...prev,
                    category: category
                  }));
                  setPage(1);
                }}
                title={`Filter by "${category}" category`}
              >
                {category}
              </button>
            ))}
          </div>
        </div>

        {/* Advanced Search Filter Panel */}
        {showFilters && (
          <div style={{
            backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',
            borderRadius: '8px',
            boxShadow: '0 2px 8px rgba(0, 0, 0, 0.05)',
            padding: '20px 24px',
            marginBottom: '20px',
            marginTop: '12px',
            animation: 'fadeIn 0.2s ease',
            border: `1px solid ${isDarkTheme ? '#2D3748' : '#E2E8F0'}`
          }}>
            {/* Filter by Tags section */}
            <div style={{
              marginBottom: '20px'
            }}>
              <h3 style={{
                fontSize: '14px',
                fontWeight: '600',
                marginBottom: '12px',
                color: isDarkTheme ? '#E2E8F0' : '#4A5568'
              }}>
                Filter by Tags
              </h3>
              
              <div style={{
                display: 'flex',
                flexWrap: 'wrap',
                gap: '8px'
              }}>
                {allTags.map(tag => (
                  <div 
                    key={`filter-tag-${tag}`}
                    style={{
                      fontSize: '13px',
                      padding: '6px 12px',
                      borderRadius: '20px',
                      backgroundColor: selectedTags.includes(tag) 
                        ? (isDarkTheme ? '#4C51BF' : '#9F7AEA') 
                        : (isDarkTheme ? '#2D3748' : '#EBF8FF'),
                      color: selectedTags.includes(tag)
                        ? (isDarkTheme ? '#E9D8FD' : '#FFFFFF')
                        : (isDarkTheme ? '#E2E8F0' : '#2C5282'),
                      cursor: 'pointer',
                      transition: 'all 0.2s ease',
                      border: `1px solid ${selectedTags.includes(tag) 
                        ? (isDarkTheme ? '#6B46C1' : '#805AD5') 
                        : (isDarkTheme ? '#4A5568' : '#BEE3F8')}`,
                      fontWeight: selectedTags.includes(tag) ? '500' : '400'
                    }}
                    onClick={() => toggleTagFilter(tag)}
                  >
                    {tag}
                  </div>
                ))}
                
                {allTags.length === 0 && (
                  <div style={{
                    color: isDarkTheme ? '#A0AEC0' : '#718096',
                    fontSize: '13px',
                    padding: '6px 0'
                  }}>
                    No tags available
                  </div>
                )}
              </div>
            </div>
            
            {/* Filter controls row */}
            <div style={{
              display: 'flex',
              flexWrap: 'wrap',
              gap: '20px',
              alignItems: 'center',
              marginBottom: '20px'
            }}>
              {/* Only Favorites checkbox */}
              <div style={{
                display: 'flex',
                alignItems: 'center',
                gap: '8px'
              }}>
                <input 
                  type="checkbox" 
                  id="only-favorites"
                  checked={searchFilters.onlyFavorites}
                  onChange={(e) => {
                    setSearchFilters(prev => ({
                      ...prev, 
                      onlyFavorites: e.target.checked
                    }));
                    setPage(1);
                  }}
                  style={{
                    width: '16px',
                    height: '16px',
                    accentColor: isDarkTheme ? '#4C51BF' : '#805AD5'
                  }}
                />
                <label 
                  htmlFor="only-favorites"
                  style={{
                    fontSize: '14px',
                    color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                    fontWeight: '500',
                    cursor: 'pointer'
                  }}
                >
                  Favorites
                </label>
              </div>
              
              {/* Category dropdown */}
              <div style={{
                display: 'flex',
                alignItems: 'center',
                gap: '8px'
              }}>
                <span style={{
                  fontSize: '14px',
                  color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                  fontWeight: '500'
                }}>
                  Category:
                </span>
                <select
                  style={{
                    padding: '4px 6px',
                    borderRadius: '6px',
                    border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
                    backgroundColor: isDarkTheme ? '#2D3748' : '#FFFFFF',
                    color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                    fontSize: '14px',
                    fontWeight: '400',
                    appearance: 'none',
                    backgroundImage: `url("data:image/svg+xml,%3Csvg xmlns='http://www.w3.org/2000/svg' width='12' height='12' viewBox='0 0 12 12' fill='none'%3E%3Cpath d='M2.5 4.5L6 8L9.5 4.5' stroke='%23718096' stroke-width='1.5' stroke-linecap='round' stroke-linejoin='round'/%3E%3C/svg%3E")`,
                    backgroundRepeat: 'no-repeat',
                    backgroundPosition: 'right 12px center',
                    paddingRight: '32px',
                    cursor: 'pointer'
                  }}
                  value={searchFilters.category}
                  onChange={(e) => {
                    setSearchFilters(prev => ({
                      ...prev,
                      category: e.target.value
                    }));
                    setPage(1);
                  }}
                >
                  <option value="all">All Categories</option>
                  {categories.map(category => (
                    <option key={`cat-filter-${category}`} value={category}>
                      {category}
                    </option>
                  ))}
                </select>
              </div>
              
              {/* Date range filter with date pickers */}
              <div style={{
                display: 'flex',
                alignItems: 'center',
                gap: '8px'
              }}>
                <span style={{
                  fontSize: '14px',
                  color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                  fontWeight: '500'
                }}>
                  Date:
                </span>
                <div style={{ position: 'relative' }}>
                  <input 
                    type="date" 
                    style={{
                      padding: '8px 12px',
                      paddingLeft: '30px',
                      borderRadius: '6px',
                      border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
                      backgroundColor: isDarkTheme ? '#2D3748' : '#FFFFFF',
                      color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                      fontSize: '14px',
                      width: '135px',
                      cursor: 'pointer',
                      fontFamily: 'inherit',
                    }}
                    value={searchFilters.dateRange.from || ''}
                    onChange={(e) => {
                      setSearchFilters(prev => ({
                        ...prev,
                        dateRange: {
                          ...prev.dateRange,
                          from: e.target.value
                        }
                      }));
                      setPage(1);
                    }}
                  />
                </div>
                <span style={{
                  fontSize: '14px',
                  color: isDarkTheme ? '#A0AEC0' : '#718096'
                }}>
                  to
                </span>
                <div style={{ position: 'relative' }}>
                  <input 
                    type="date" 
                    style={{
                      padding: '8px 12px',
                      paddingLeft: '30px',
                      borderRadius: '6px',
                      border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
                      backgroundColor: isDarkTheme ? '#2D3748' : '#FFFFFF',
                      color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                      fontSize: '14px',
                      width: '135px',
                      cursor: 'pointer',
                      fontFamily: 'inherit',
                    }}
                    value={searchFilters.dateRange.to || ''}
                    onChange={(e) => {
                      setSearchFilters(prev => ({
                        ...prev,
                        dateRange: {
                          ...prev.dateRange,
                          to: e.target.value
                        }
                      }));
                      setPage(1);
                    }}
                  />
                </div>
              </div>
            </div>
            
            {/* Recent searches section */}
            {recentSearches.length > 0 && (
              <div style={{
                marginTop: '12px'
              }}>
                <h3 style={{
                  fontSize: '14px',
                  fontWeight: '600',
                  marginBottom: '12px',
                  color: isDarkTheme ? '#E2E8F0' : '#4A5568'
                }}>
                  Recent Searches
                </h3>
                <div style={{
                  display: 'flex',
                  flexWrap: 'wrap',
                  gap: '8px'
                }}>
                  {recentSearches.map((search, index) => (
                    <div 
                      key={`recent-search-${index}`}
                      style={{
                        display: 'flex',
                        alignItems: 'center',
                        gap: '6px',
                        padding: '4px 6px',
                        backgroundColor: isDarkTheme ? '#2D3748' : '#EDF2F7',
                        color: isDarkTheme ? '#A0AEC0' : '#718096',
                        borderRadius: '16px',
                        fontSize: '13px',
                        cursor: 'pointer',
                        transition: 'all 0.2s ease',
                        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`
                      }}
                      onClick={() => {
                        setSearchTerm(search);
                        setPage(1);
                      }}
                    >
                      <span style={{
                        fontSize: '12px',
                        color: isDarkTheme ? '#A0AEC0' : '#718096'
                      }}>
                        🕒
                      </span>
                      <span>{search}</span>
                    </div>
                  ))}
                </div>
              </div>
            )}
            
            {/* Clear filters button */}
            <div style={{
              display: 'flex',
              justifyContent: 'flex-end',
              marginTop: '16px'
            }}>
              <button 
                style={{
                  padding: '8px 16px',
                  backgroundColor: isDarkTheme ? '#2D3748' : '#E2E8F0',
                  color: isDarkTheme ? '#A0AEC0' : '#4A5568',
                  border: 'none',
                  borderRadius: '6px',
                  cursor: 'pointer',
                  fontSize: '14px',
                  fontWeight: '500',
                  transition: 'all 0.2s ease'
                }}
                onClick={() => {
                  setSelectedTags([]);
                  setSearchFilters({
                    onlyFavorites: false,
                    dateRange: { from: null, to: null },
                    contentType: 'all',
                    category: 'all'
                  });
                  setPage(1);
                }}
              >
                Clear All Filters
              </button>
            </div>
          </div>
        )}
        
        <div style={styles.searchResultStats}>
          Found {filteredPrompts.length} prompt{filteredPrompts.length !== 1 && 's'}
          {(
            selectedTags.length > 0 ||
            searchFilters.category !== 'all' ||
            searchFilters.onlyFavorites ||
            hasDateFilter
          ) && ' with applied filters'}
        </div>

        {/* Active Filters Pills */}
        {(
          selectedTags.length > 0 ||
          searchFilters.category !== 'all' ||
          searchFilters.onlyFavorites ||
          hasDateFilter
        ) && (
          <div style={styles.activeFiltersContainer}>
            {/* Tag filters */}
            {selectedTags.map(tag => (
              <div key={`filter-${tag}`} style={styles.activeFilter}>
                <span style={styles.activeFilterLabel}>{tag}</span>
                <span
                  style={styles.activeFilterRemove}
                  onClick={() => {
                    toggleTagFilter(tag);
                    setPage(1);
                  }}
                >
                  ×
                </span>
              </div>
            ))}

            {/* Category filter */}
            {searchFilters.category !== 'all' && (
              <div style={styles.activeFilter}>
                <span style={styles.activeFilterLabel}>
                  Category: {searchFilters.category}
                </span>
                <span
                  style={styles.activeFilterRemove}
                  onClick={() => {
                    setSearchFilters(prev => ({ ...prev, category: 'all' }));
                    setPage(1);
                  }}
                >
                  ×
                </span>
              </div>
            )}

            {/* Favorites toggle */}
            {searchFilters.onlyFavorites && (
              <div style={styles.activeFilter}>
                <span style={styles.activeFilterLabel}>Favorites</span>
                <span
                  style={styles.activeFilterRemove}
                  onClick={() => {
                    setSearchFilters(prev => ({ ...prev, onlyFavorites: false }));
                    setPage(1);
                  }}
                >
                  ×
                </span>
              </div>
            )}

            {/* Date-range filter */}
            {hasDateFilter && (
              <div key="filter-date" style={styles.activeFilter}>
                <span style={styles.activeFilterLabel}>
                  Date:{' '}
                  {new Date(searchFilters.dateRange.from).toLocaleDateString()} –{' '}
                  {new Date(searchFilters.dateRange.to).toLocaleDateString()}
                </span>
                <span
                  style={styles.activeFilterRemove}
                  onClick={() => {
                    setSearchFilters(prev => ({ ...prev, dateRange: null }));
                    setPage(1);
                  }}
                >
                  ×
                </span>
              </div>
            )}
          </div>
        )}

        {/* Success or Error Message */}
        {error && (
          <div style={styles.errorMessage}>
            {error}
          </div>
        )}

        {success && (
          <div style={styles.successMessage}>
            {success}
          </div>
        )}
        
        {showForm && (
        <div style={modalStyles.overlay}>
          <div style={modalStyles.modal}>
            <div style={modalStyles.modalHeader}>
              <h2 style={modalStyles.modalTitle}>New Prompt</h2>
              <button 
                style={modalStyles.closeButton}
                onClick={() => setShowForm(false)}
              >
                ✕
              </button>
            </div>
            <form onSubmit={handleSubmit}>
              <div style={modalStyles.formGroup}>
                <label style={modalStyles.label} htmlFor="title">Title*</label>
                <input 
                  type="text"
                  id="title"
                  name="title"
                  value={formData.title}
                  onChange={handleInputChange}
                  style={modalStyles.input}
                  placeholder="Enter a title for your prompt"
                  required
                />
              </div>
              
              <div style={modalStyles.formGroup}>
                <label style={modalStyles.label} htmlFor="description">Description</label>
                <input
                  type="text"
                  id="description"
                  name="description" 
                  value={formData.description}
                  onChange={handleInputChange}
                  style={modalStyles.input}
                  placeholder="Brief description of what this prompt does"
                />
              </div>
              
              <div style={modalStyles.formGroup}>
                <label style={modalStyles.label} htmlFor="prompt">Prompt*</label>
                <textarea
                  id="prompt"
                  name="prompt"
                  value={formData.prompt}
                  onChange={handleInputChange}
                  style={modalStyles.textarea}
                  placeholder="Enter your prompt text here..."
                  required
                />
              </div>
              
              {/* Tag input with pills */}
              <div style={modalStyles.formGroup}>
                <label style={modalStyles.label} htmlFor="tagInput">Tags</label>
                <div style={modalStyles.inputWrapper}>
                  <input
                    id="tagInput"
                    value={tagInput}
                    onChange={handleTagInputChange}
                    onKeyDown={(e) => {
                      if (e.key === 'Enter' && tagInput.trim()) {
                        e.preventDefault();
                        addTag(tagInput.trim());
                      }
                    }}
                    style={modalStyles.input}
                    placeholder="Type a tag and press Enter"
                  />
                  {showTagSuggestions && tagSuggestions.length > 0 && (
                    <ul style={styles.tagSuggestionContainer}>
                      {tagSuggestions.map(tag => (
                        <li
                          key={tag}
                          style={styles.tagSuggestionItem}
                          onMouseDown={() => addTag(tag)}
                        >
                          {tag}
                        </li>
                      ))}
                    </ul>
                  )}
                </div>
                <div style={modalStyles.tagPillsContainer}>
                  {formData.tags && formData.tags.split(',').map(tag => tag.trim()).filter(tag => tag).map((tag, index) => (
                    <div key={`form-tag-${index}`} style={modalStyles.tagPill}>
                      {tag}
                      <button 
                        type="button"
                        style={modalStyles.removeTagButton}
                        onClick={() => {
                          const updatedTags = formData.tags
                            .split(',')
                            .map(t => t.trim())
                            .filter(t => t && t !== tag)
                            .join(', ');
                          
                          setFormData(prev => ({ ...prev, tags: updatedTags }));
                        }}
                      >
                        ×
                      </button>
                    </div>
                  ))}
                </div>
              </div>
              
              {/* Category with gear icon */}
              <div style={modalStyles.formGroup}>
                <label style={modalStyles.label} htmlFor="category">Category</label>
                <div style={modalStyles.categoryRow}>
                  <select
                    id="category"
                    name="category"
                    value={isNewCategory ? "__new__" : formData.category}
                    onChange={(e) => {
                      if (e.target.value === "__new__") {
                        setIsNewCategory(true);
                      } else {
                        setIsNewCategory(false);
                        setFormData(prev => ({ ...prev, category: e.target.value }));
                      }
                    }}
                    style={modalStyles.select}
                  >
                    {categories.map(category => (
                      <option key={category} value={category}>
                        {category}
                      </option>
                    ))}
                    <option value="__new__">+ Create New Category</option>
                  </select>
                  {categories.length > 1 && !isNewCategory && !editingCategory && (
                    <div 
                      style={{
                        ...modalStyles.gearIcon,
                        opacity: 1,
                        cursor: 'pointer'
                      }}
                      onClick={() => {
                        setShowCategoryManagement(!showCategoryManagement);
                      }}
                      title="Manage Categories"
                    >
                      <SvgGear />
                    </div>
                    )}
                </div>

                    {/* Category Management UI in modal - shown when toggled */}
                    {showCategoryManagement && (
                      <div style={{
                        marginTop: '12px',
                        maxHeight: '200px',
                        overflowY: 'auto',
                        padding: '8px',
                        border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
                        borderRadius: '6px',
                        backgroundColor: isDarkTheme ? '#1A202C' : '#F7FAFC',
                        animation: 'fadeIn 0.2s ease'
                      }}>
                        <div style={{
                          fontSize: '13px',
                          marginBottom: '8px',
                          padding: '4px',
                          borderBottom: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
                          fontWeight: '500',
                          color: isDarkTheme ? '#E2E8F0' : '#4A5568'
                        }}>
                          Manage Categories
                        </div>
                        
                        {/* Display General category (read-only) */}
                        <div 
                          style={{
                            display: 'flex',
                            justifyContent: 'space-between',
                            alignItems: 'center',
                            padding: '6px 8px',
                            cursor: 'default',
                            borderRadius: '4px',
                            marginBottom: '4px',
                            backgroundColor: hoveredCategory === 'General' ? 
                              (isDarkTheme ? '#2D3748' : '#F7FAFC') : 'transparent',
                            opacity: 0.8,
                            borderLeft: `3px solid ${getCategoryColor('General')}`,
                          }}
                          onMouseEnter={() => setHoveredCategory('General')}
                          onMouseLeave={() => setHoveredCategory(null)}
                        >
                          <span style={{
                            color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                            fontWeight: '600'
                          }}>
                            General (Default)
                          </span>
                          <div style={{
                            padding: '2px 6px',
                            fontSize: '12px',
                            color: isDarkTheme ? '#718096' : '#A0AEC0',
                            fontStyle: 'italic'
                          }}>
                            Cannot be modified
                          </div>
                        </div>
                        
                        {/* Categories list - excluding General */}
                        {categories
                          .filter(category => category !== 'General')
                          .map(category => (
                            <div 
                              key={`manage-modal-${category}`}
                              style={{
                                display: 'flex',
                                justifyContent: 'space-between',
                                alignItems: 'center',
                                padding: '6px 8px',
                                cursor: 'default',
                                borderRadius: '4px',
                                marginBottom: '4px',
                                backgroundColor: hoveredCategory === category ? 
                                  (isDarkTheme ? '#2D3748' : '#F7FAFC') : 'transparent',
                                position: 'relative',
                                borderLeft: `3px solid ${getCategoryColor(category)}`,
                              }}
                              onMouseEnter={() => setHoveredCategory(category)}
                              onMouseLeave={() => setHoveredCategory(null)}
                            >
                              {/* Inline editing - show input field when this category is being edited */}
                              {editingCategory === category ? (
                                <div style={{
                                  display: 'flex',
                                  alignItems: 'center',
                                  gap: '8px',
                                  flex: 1,
                                  marginRight: '8px'
                                }}>
                                 {/* Color circle */}
                                  <div 
                                    style={{
                                      width: '20px',
                                      height: '20px',
                                      borderRadius: '50%',
                                      backgroundColor: editingCategoryColor === category ? selectedColor : getCategoryColor(category),
                                      border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
                                      cursor: 'pointer',
                                      transition: 'all 0.2s ease',
                                      boxShadow: '0 1px 3px rgba(0,0,0,0.2)',
                                      marginRight: '8px',  // Add margin to separate from category name
                                      position: 'relative',  // For the tooltip/hint
                                      '&:hover': {
                                        transform: 'scale(1.1)',
                                        boxShadow: '0 2px 5px rgba(0,0,0,0.3)'
                                      },
                                      '&:after': {  // Small hint indicator
                                        content: '""',
                                        position: 'absolute',
                                        right: '-2px',
                                        bottom: '-2px',
                                        width: '8px',
                                        height: '8px',
                                        borderRadius: '50%',
                                        backgroundColor: isDarkTheme ? '#A0AEC0' : '#4A5568',
                                        border: `1px solid ${isDarkTheme ? '#1A202C' : 'white'}`,
                                      }
                                    }}
                                    onClick={() => {
                                      if (editingCategoryColor === category) {
                                        // Close color picker if it's already open
                                        setEditingCategoryColor(null);
                                        setSelectedColor('');
                                      } else {
                                        // Open color picker for this category
                                        setEditingCategoryColor(category);
                                        setSelectedColor(getCategoryColor(category));
                                      }
                                    }}
                                    title="Click to change category color"
                                  ></div>
                                  
                                  {/* Color picker - only appears when the color circle is clicked */}
                                  {editingCategoryColor === category && (
                                    <div style={{
                                      position: 'absolute',
                                      top: '100%',
                                      left: '0',
                                      zIndex: 10,
                                      display: 'flex',
                                      flexWrap: 'wrap',
                                      gap: '6px',
                                      width: '184px', // 8 colors per row × (16px + 6px gap)
                                      padding: '8px',
                                      backgroundColor: isDarkTheme ? '#1A202C' : 'white',
                                      borderRadius: '6px',
                                      boxShadow: '0 4px 12px rgba(0, 0, 0, 0.15)',
                                      border: `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
                                      marginTop: '8px'
                                    }}>
                                      {/* Color options - including both dark and light theme options */}
                                      {[
                                        // Blues
                                        '#3182CE', '#63B3ED', 
                                        // Greens
                                        '#38A169', '#68D391', 
                                        // Purples
                                        '#805AD5', '#B794F4', 
                                        // Yellows
                                        '#D69E2E', '#F6E05E', 
                                        // Oranges
                                        '#DD6B20', '#F6AD55', 
                                        // Reds
                                        '#E53E3E', '#FC8181', 
                                        // Teals
                                        '#2C7A7B', '#4FD1C5', 
                                        // Pinks
                                        '#D53F8C', '#F687B3',
                                        // Grays (for more subtle categories)
                                        '#4A5568', '#A0AEC0',
                                        // Other colors
                                        '#9C4221', '#C05621',
                                        '#702459', '#B83280',
                                        '#2A4365', '#2B6CB0'
                                      ].map((color, index) => (
                                        <div
                                          key={`color-${index}`}
                                          style={{
                                            width: '16px',
                                            height: '16px',
                                            borderRadius: '50%',
                                            backgroundColor: color,
                                            border: selectedColor === color 
                                              ? `2px solid ${isDarkTheme ? 'white' : 'black'}`
                                              : `1px solid ${isDarkTheme ? '#4A5568' : '#E2E8F0'}`,
                                            cursor: 'pointer',
                                            transition: 'transform 0.1s ease',
                                            '&:hover': {
                                              transform: 'scale(1.2)'
                                            }
                                          }}
                                          onClick={() => setSelectedColor(color)}
                                          title={color}
                                        ></div>
                                        
                                      ))}
                                    </div>
                                  )}
                                  <input
                                    type="text"
                                    value={editCategoryName}
                                    onChange={(e) => setEditCategoryName(e.target.value)}
                                    style={{
                                      padding: '4px 8px',
                                      borderRadius: '4px',
                                      border: `1px solid ${isDarkTheme ? '#4A5568' : '#CBD5E0'}`,
                                      backgroundColor: isDarkTheme ? '#1A202C' : 'white',
                                      color: isDarkTheme ? '#E2E8F0' : 'inherit',
                                      fontSize: '13px',
                                      flex: 1
                                    }}
                                    autoFocus
                                  />
                                  <button
                                    type="button"
                                    style={{
                                      padding: '4px 8px',
                                      backgroundColor: isDarkTheme ? '#4C51BF' : '#4299E1',
                                      color: 'white',
                                      border: 'none',
                                      borderRadius: '4px',
                                      cursor: 'pointer',
                                      fontSize: '12px',
                                      fontWeight: '500'
                                    }}
                                    onClick={handleEditCategory}
                                   disabled={!(nameChanged || colorChanged)}
                                  >
                                    Save
                                  </button>
                                  <button
                                    type="button"
                                    style={{
                                      padding: '4px 8px',
                                      backgroundColor: isDarkTheme ? '#4A5568' : '#E2E8F0',
                                      color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                                      border: 'none',
                                      borderRadius: '4px',
                                      cursor: 'pointer',
                                      fontSize: '12px'
                                    }}
                                    onClick={() => {
                                      setEditingCategory(null);
                                      setEditCategoryName('');
                                    }}
                                  >
                                    Cancel
                                  </button>
                                </div>
                              ) : (
                                <>
                                  <span style={{
                                    color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                                    fontWeight: '400'
                                  }}>
                                    {category}
                                  </span>
                                  <div style={{
                                    display: 'flex',
                                    gap: '6px',
                                    opacity: hoveredCategory === category ? 1 : 0.5
                                  }}>
                                    <button
                                      type="button"
                                      style={{
                                        backgroundColor: 'transparent',
                                        border: 'none',
                                        cursor: 'pointer',
                                        padding: '2px 6px',
                                        fontSize: '12px',
                                        color: isDarkTheme ? '#A0AEC0' : '#718096',
                                        borderRadius: '3px',
                                        ...(hoveredCategory === category ? {
                                          backgroundColor: isDarkTheme ? '#2D3748' : '#EBF8FF',
                                          color: isDarkTheme ? '#E2E8F0' : '#3182CE'
                                        } : {})
                                      }}
                                      onClick={() => {
                                        setEditingCategory(category);
                                        setEditCategoryName(category);
                                      }}
                                      title="Edit category"
                                    >
                                      ✏️ Edit
                                    </button>
                                    <button
                                      type="button"
                                      style={{
                                        backgroundColor: 'transparent',
                                        border: 'none',
                                        cursor: 'pointer',
                                        padding: '2px 6px',
                                        fontSize: '12px',
                                        color: isDarkTheme ? '#A0AEC0' : '#718096',
                                        borderRadius: '3px',
                                        ...(hoveredCategory === category ? {
                                          backgroundColor: isDarkTheme ? '#742A2A' : '#FED7D7',
                                          color: isDarkTheme ? '#FEB2B2' : '#9B2C2C'
                                        } : {})
                                      }}
                                      onClick={() => handleDeleteCategory(category)}
                                      title="Delete category"
                                    >
                                      🗑️ Delete
                                    </button>
                                  </div>
                                </>
                              )}
                            </div>
                          ))
                        }
                        
                        {/* Add new category option */}
                        <button
                          type="button"
                          style={{
                            display: 'flex',
                            alignItems: 'center',
                            gap: '6px',
                            width: '100%',
                            padding: '8px',
                            marginTop: '8px',
                            backgroundColor: isDarkTheme ? '#2D3748' : '#E2E8F0',
                            color: isDarkTheme ? '#E2E8F0' : '#4A5568',
                            border: 'none',
                            borderRadius: '4px',
                            cursor: 'pointer',
                            fontSize: '13px',
                            justifyContent: 'center',
                            fontWeight: '500',
                            transition: 'all 0.2s ease',
                            ':hover': {
                              backgroundColor: isDarkTheme ? '#4A5568' : '#CBD5E0',
                            }
                          }}
                          onClick={() => {
                            setIsNewCategory(true);
                            setShowCategoryManagement(false);
                          }}
                        >
                          <span style={{ fontSize: '14px' }}>+</span>
                          <span>Add New Category</span>
                        </button>
                      </div>
                    )}
                
                {/* New category input */}
                {isNewCategory && (
                  <div style={{marginTop: '10px'}}>
                    <input
                      type="text"
                      value={newCategoryName}
                      onChange={(e) => setNewCategoryName(e.target.value)}
                      style={modalStyles.input}
                      placeholder="Enter new category name"
                      autoFocus
                    />
                    <div style={{display: 'flex', gap: '8px', marginTop: '8px'}}>
                      <button
                        type="button"
                        style={{
                          ...modalStyles.submitButton,
                          padding: '6px 12px',
                          fontSize: '13px'
                        }}
                        onClick={handleCreateCategory}
                        disabled={!newCategoryName.trim()}
                      >
                        Create
                      </button>
                      <button
                        type="button"
                        style={{
                          ...modalStyles.cancelButton,
                          padding: '6px 12px',
                          fontSize: '13px'
                        }}
                        onClick={() => {
                          setIsNewCategory(false);
                          setNewCategoryName('');
                        }}
                      >
                        Cancel
                      </button>
                    </div>
                  </div>
                )}
                
                {/* Favorite checkbox with star icon */}
                <div style={modalStyles.formGroup}>
                  <div style={modalStyles.checkboxContainer}>
                    <input
                      type="checkbox"
                      id="favorite"
                      name="favorite"
                      checked={formData.favorite}
                      onChange={handleInputChange}
                      style={modalStyles.checkbox}
                    />
                    <label style={modalStyles.checkboxLabel} htmlFor="favorite">
                      <SvgStar
                        width="16"
                        height="16"
                        style={{
                          color: formData.favorite ? 
                            (isDarkTheme ? '#F6E05E' : '#F6AD55') : 
                            (isDarkTheme ? '#A0AEC0' : '#CBD5E0')
                        }}
                      />
                      Add to Favorites
                    </label>
                  </div>
                </div>
              </div>
              
              <div style={modalStyles.formActions}>
                <button 
                  type="button" 
                  style={modalStyles.cancelButton}
                  onClick={() => setShowForm(false)}
                >
                  Cancel
                </button>
                <button 
                  type="submit" 
                  style={modalStyles.submitButton}
                  disabled={isSubmitting}
                >
                  {isSubmitting ? 'Saving...' : 'Save Prompt'}
                </button>
              </div>
            </form>
          </div>
        </div>
      )}
        
        {/* Tabs Navigation */}
        <div style={tabStyles.tabsContainer}>
          {/* Favorites tab */}
          <button
            style={{
              ...tabStyles.tabButton,
              ...(activeTab === 'favorites' ? tabStyles.activeTabButton : {})
            }}
            onClick={() => setActiveTab('favorites')}
          >
            <span style={tabStyles.tabIcon}>
              <SvgStar width="18" height="18" style={{ verticalAlign: 'middle', marginRight: '8px' }} />
            </span>
            Favorites ({favoritePrompts.length})
          </button>

          {/* Recently Used tab */}
          <button
            style={{
              ...tabStyles.tabButton,
              ...(activeTab === 'recent' ? tabStyles.activeTabButton : {})
            }}
            onClick={() => setActiveTab('recent')}
          >
            <span style={tabStyles.tabIcon}>
              <SvgHourglass width="18" height="18" style={{ verticalAlign: 'middle', marginRight: '8px' }} />
            </span>
            Recently Used ({recentlyUsedPrompts.length})
          </button>

          {/* All Prompts tab */}
          <button
            style={{
              ...tabStyles.tabButton,
              ...(activeTab === 'all' ? tabStyles.activeTabButton : {})
            }}
            onClick={() => setActiveTab('all')}
          >
            <span style={tabStyles.tabIcon}>
              <SvgClipboard width="18" height="18" style={{ verticalAlign: 'middle', marginRight: '8px' }} />
            </span>
            All Prompts ({filteredPrompts.length})
          </button>
        </div>

        {/* Tab Content */}
        <div style={tabStyles.tabContent}>
          {/* Favorites Tab Content */}
          {activeTab === 'favorites' && (
            <div>
              {categoryFilteredFavorites.length > 0 ? (
                <div style={styles.favoritesScrollContainer}>
                  {/* Left arrow */}
                  <div
                    style={{
                      ...styles.scrollIndicatorLeft,
                      display: 'flex',
                      alignItems: 'center',
                      justifyContent: 'center'
                    }}
                    id="scroll-arrow-left"
                    onClick={() => {
                      const c = document.getElementById('favorites-scroll-container');
                      if (c) c.scrollBy({ left: -300, behavior: 'smooth' });
                    }}
                  >
                    <div style={{
                      display: 'flex',
                      alignItems: 'center',
                      justifyContent: 'center',
                      width: '100%',
                      height: '100%',
                      fontSize: '18px',
                      fontFamily: 'monospace',
                      position: 'relative',
                      right: '2px',
                      top: '0px'
                    }}>
                      ◀
                    </div>
                  </div>

                  {/* The two-row grid */}
                  <div
                    id="favorites-scroll-container"
                    style={{
                      ...styles.favoritesScroll,
                      flexWrap: 'wrap',
                      justifyContent: 'flex-start',
                      gap: '24px',
                      paddingLeft: '20px',
                      paddingRight: '20px'
                    }}
                  >
                    {categoryFilteredFavorites.map((prompt, index) => {
                      const promptText = typeof prompt.prompt === 'string'
                        ? prompt.prompt
                        : JSON.stringify(prompt.prompt);
                      const promptPreview = promptText
                        .split('\n')
                        .slice(0, 3)
                        .join('\n');

                      const isDragging = draggedIndex === index;
                      const isDraggedOver = dragOverIndex === index;
                      const cardStyle = {
                        ...styles.card,
                        ...(isDragging ? styles.cardDragging : {}),
                        ...(isDraggedOver ? styles.cardDragOver : {}),
                      };

                      return (
                        <div
                          key={`favorite-${index}-${prompt.title}`}
                          draggable
                          onDragStart={e => handleDragStart(e, index)}
                          onDragOver={e => handleDragOver(e, index)}
                          onDrop={e => handleDrop(e, index)}
                          onDragEnd={handleDragEnd}
                          style={cardStyle}
                          onClick={() => openPromptNote(prompt)}
                        >
                          <div
                            style={{
                              ...styles.dragHandle,
                              ...(isDragging ? styles.dragHandleActive : {})
                            }}
                            title="Drag to reorder"
                          >⋮⋮</div>

                          <div style={styles.cardContentWrapper}>
                            <div style={styles.cardTitleSection}>
                              <div style={styles.cardTitle}>
                                {prompt.title || 'Untitled'}
                              </div>
                            </div>

                            <div style={styles.cardDescriptionSection}>
                              {prompt.description ? (
                                <div style={styles.cardDescription}>
                                  {prompt.description}
                                </div>
                              ) : (
                                <div style={{ ...styles.cardDescription, fontStyle: 'italic', opacity: 0.7 }}>
                                  No description
                                </div>
                              )}
                            </div>

                            <div style={styles.cardPromptPreviewSection}>
                              <div style={styles.cardPromptPreview}>
                                {promptPreview.length > 0
                                  ? truncateText(promptPreview, 150)
                                  : <span style={{ fontStyle: 'italic', opacity: 0.7 }}>No preview available</span>}
                              </div>
                            </div>

                            <div style={styles.cardTagsSection}>
                              <div style={styles.cardTags}>
                                {formatTags(prompt.tags).map((tag, tagIndex) => (
                                  <span
                                    key={`tag-${tagIndex}`}
                                    style={styles.tag}
                                    onClick={e => {
                                      e.stopPropagation();
                                      toggleTagFilter(tag);
                                    }}
                                  >
                                    {tag}
                                  </span>
                                ))}
                                {formatTags(prompt.tags).length === 0 && (
                                  <span style={{ 
                                    fontStyle: 'italic', 
                                    opacity: 0.7, 
                                    color: isDarkTheme ? '#A0AEC0' : '#94A3B8'
                                  }}>
                                    No tags
                                  </span>
                                )}
                              </div>
                            </div>

                            <div style={styles.cardActionsSection}>
                              <div style={styles.cardActions}>
                                <button
                                  style={styles.cardButton}
                                  onClick={e => {
                                    e.stopPropagation();
                                    copyPromptToClipboard(prompt);
                                  }}
                                  title="Copy prompt to clipboard"
                                >
                                  <SvgClipboard />
                                </button>
                                <button
                                  style={styles.cardButton}
                                  onClick={e => {
                                    e.stopPropagation();
                                    toggleFavorite(prompt);
                                  }}
                                  title="Remove from favorites"
                                >
                                  <SvgStar />
                                </button>
                              </div>
                            </div>
                            <div 
                              style={{
                                ...styles.categoryBadge,
                                backgroundColor: getCategoryColor(prompt.category),
                                color: isDarkTheme 
                                  ? 'rgba(255, 255, 255, 0.9)' 
                                  : 'rgba(0, 0, 0, 0.8)'
                              }}
                              title={`Category: ${prompt.category || 'General'}`}
                            >
                              {prompt.category || 'General'}
                            </div>
                          </div>
                        </div>
                      );
                    })}
                  </div>

                  {/* Right arrow */}
                  <div
                    style={{
                      ...styles.scrollIndicatorRight,
                      display: 'flex',
                      alignItems: 'center',
                      justifyContent: 'center'
                    }}
                    id="scroll-arrow-right"
                    onClick={() => {
                      const c = document.getElementById('favorites-scroll-container');
                      if (c) c.scrollBy({ left: 300, behavior: 'smooth' });
                    }}
                  >
                    <div style={{
                      display: 'flex',
                      alignItems: 'center',
                      justifyContent: 'center',
                      width: '100%',
                      height: '100%',
                      fontSize: '18px',
                      fontFamily: 'monospace',
                      position: 'relative',
                      left: '2px',
                      top: '0px'
                    }}>
                      ▶
                    </div>
                  </div>
                </div>
              ) : (
                <div style={styles.emptyState}>
                  {searchFilters.category !== 'all' ? (
                    <p style={styles.emptyStateText}>
                      No favorite prompts in the "{searchFilters.category}" category
                    </p>
                  ) : (
                    <p style={styles.emptyStateText}>No favorite prompts found</p>
                  )}
                  <button
                    style={styles.addButton}
                    onClick={() => setShowForm(true)}
                  >
                    + New Prompt
                  </button>
                </div>
              )}
            </div>
          )}

          {/* Recently Used Tab Content */}
          {activeTab === 'recent' && (
            <div>
              {renderRecentlyUsedSection()}
            </div>
          )}

          {/* All Prompts Tab Content */}
          {activeTab === 'all' && (
            <div style={styles.allPromptsSection}>
              {filteredPrompts.length > 0 ? (
                <>
                  {/* Batch actions bar */}
                  {renderBatchActionBar()}
                  
                  {/* Batch confirm dialog */}
                  {renderBatchConfirmDialog()}
                  
                  {/* Table container */}
                  <div style={styles.tableContainer}>
                    <table style={styles.table}>
                      {renderTableHeader()}
                      <tbody>
                        {paginatedPrompts.map((prompt, index) => (
                          <tr
                            key={prompt.$path}
                            style={styles.tableRow(index)}
                            onMouseEnter={e => e.currentTarget.style.backgroundColor = styles.tableRowHover.backgroundColor}
                            onMouseLeave={e => e.currentTarget.style.backgroundColor = 'transparent'}
                          >
                            <td style={{ ...styles.tableCell, width: '20%' }}>{prompt.title}</td>
                            <td style={{ ...styles.tableCell, width: '30%' }}>
                              {prompt.description ? truncateText(prompt.description, 60) : '—'}
                            </td>
                            <td style={{ ...styles.tableCell, width: '12%' }}>
                              {/* Format category as badge with larger text */}
                              <div 
                                style={{
                                  display: 'inline-block',
                                  padding: '5px 9px',  // Slightly increased padding
                                  borderRadius: '4px',
                                  fontSize: '13px',    // Increased from 11px
                                  fontWeight: '600',
                                  textTransform: 'uppercase',
                                  letterSpacing: '0.5px',
                                  backgroundColor: getCategoryColor(prompt.category || 'General'),
                                  color: isDarkTheme 
                                    ? 'rgba(255, 255, 255, 0.9)' 
                                    : 'rgba(0, 0, 0, 0.8)',
                                  /* allow full display */
                                  whiteSpace: 'normal',
                                  overflow: 'visible',
                                  textOverflow: 'clip'
                                }}
                              >
                                {prompt.category || 'General'}
                              </div>
                            </td>
                            <td style={{ ...styles.tableCell, width: '18%' }}>
                              <div style={styles.tableTagContainer}>
                                {formatTags(prompt.tags).map((tag, i) => (
                                  <span
                                    key={`table-tag-${i}-${tag}`}
                                    style={styles.tableTag}
                                    onClick={(e) => {
                                      e.stopPropagation();
                                      toggleTagFilter(tag);
                                    }}
                                  >
                                    {tag}
                                  </span>
                                ))}
                                {formatTags(prompt.tags).length === 0 && (
                                  <span style={{ 
                                    color: isDarkTheme ? '#718096' : '#A0AEC0', 
                                    fontSize: '13px',
                                    fontStyle: 'italic' 
                                  }}>
                                    —
                                  </span>
                                )}
                              </div>
                            </td>
                            <td style={{ ...styles.tableCell, width: '12%' }}>{new Date(prompt.created).toLocaleDateString()}</td>
                            <td style={{ ...styles.tableCell, width: '8%', textAlign: 'right' }}>
                              <div style={styles.tableActions}>
                                <button
                                  style={styles.actionButton}
                                  onClick={(e) => {
                                    e.stopPropagation();
                                    copyPromptToClipboard(prompt);
                                  }}
                                  title="Copy prompt"
                                >
                                  <SvgClipboard />
                                </button>
                                <button
                                  style={styles.actionButton}
                                  onClick={(e) => {
                                    e.stopPropagation();
                                    toggleFavorite(prompt);
                                  }}
                                  title={prompt.favorite ? "Remove from favorites" : "Add to favorites"}
                                >
                                  {prompt.favorite ? <SvgStar /> : <SvgStarOff />}
                                </button>
                              </div>
                            </td>
                          </tr>
                        ))}
                      </tbody>
                    </table>
                  </div>
                  
                  {/* Pagination controls */}
                  <div style={styles.pagination}>
                    <div style={styles.paginationInfo}>
                      Page {page} of {totalPages}
                    </div>
                    <div style={styles.paginationButtons}>
                      <button
                        style={styles.pageButton}
                        onClick={prevPage}
                        disabled={page === 1}
                      >
                        Prev
                      </button>
                      {Array.from({ length: Math.min(totalPages, 5) }, (_, i) => {
                        let pageNumber;
                        if (totalPages <= 5) {
                          pageNumber = i + 1;
                        } else if (page <= 3) {
                          pageNumber = i + 1;
                        } else if (page >= totalPages - 2) {
                          pageNumber = totalPages - 4 + i;
                        } else {
                          pageNumber = page - 2 + i;
                        }
                        
                        return (
                          <button
                            key={pageNumber}
                            style={{
                              ...styles.pageButton,
                              ...(page === pageNumber ? styles.activePageButton : {})
                            }}
                            onClick={() => handlePageChange(pageNumber)}
                          >
                            {pageNumber}
                          </button>
                        );
                      })}
                      <button
                        style={styles.pageButton}
                        onClick={nextPage}
                        disabled={page === totalPages}
                      >
                        Next
                      </button>
                    </div>
                  </div>
                </>
              ) : (
                <div style={styles.emptyState}>
                  <p style={styles.emptyStateText}>No prompts to show yet.</p>
                </div>
              )}
            </div>
          )}
        </div>
              {/* Floating Action Button - moved up 25px */}
              <button
                style={{
                  ...modalStyles.floatingActionButton,
                  bottom: '80px',  // Changed from '30px' to '55px' to move it up 25px
                  right: '50px'
                }}
                onClick={() => setShowForm(!showForm)}
                title="Add new prompt"
              >
                <SvgPlus />
              </button>
      </div>
    );
  }
      
  // If we're in Datacore environment, return the component
  return AIPromptsManager;

} catch (error) {
  // Error fallback component
  function ErrorComponent() {
    console.error("Error in AI Prompts Manager:", error);
    
    return (
      <div style={{ 
        padding: '20px', 
        backgroundColor: '#FFF5F5', 
        borderRadius: '8px', 
        margin: '20px 0',
        color: '#C53030',
        border: '1px solid #FC8181'
      }}>
        <h2 style={{ color: '#C53030', margin: '0 0 15px 0' }}>Error Loading AI Prompts Manager</h2>
        <p>There was an error loading the AI Prompts Manager component:</p>
        <pre style={{
          backgroundColor: '#FEB2B2',
          padding: '10px',
          borderRadius: '4px',
          overflow: 'auto',
          color: '#742A2A',
          fontFamily: 'monospace'
        }}>
          {error.message}
        </pre>
        <p>Please check the console for more details or try refreshing the page.</p>
      </div>
    );
  }
  
  return ErrorComponent;
}
```