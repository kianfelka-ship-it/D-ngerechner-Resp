import React, { useState, useCallback, useMemo, useEffect } from 'react';
import { AppState, Product, Phase, DetailedWaterProfile, NutrientRecipe, WeekTargets } from './types';
import { INITIAL_APP_STATE, FERTILIZER_DATABASE, DEFAULT_PRODUCT_ROW, PHASE_TARGETS, RECIPES_KEY, PERMANENT_RECIPES } from './constants';
import ProductRow from './components/ProductRow';
import NumberInput from './components/NumberInput';
import SelectInput from './components/SelectInput';
import NutrientSumDisplay from './components/NutrientSumDisplay';
import WaterAnalysisDialog from './components/WaterAnalysisDialog';
import RecipeEditorDialog from './components/RecipeEditorDialog';
import BrandSelector from './components/BrandSelector';
import ProductMultiSelector from './components/ProductMultiSelector'; // Import the new component
import { useLocalStorage } from './hooks/useLocalStorage';
import { compute } from './utils/calculator';
import { suggestDoses } from './utils/doseSolver';


const App: React.FC = () => {
  const [persistedRecipes, setPersistedRecipes] = useLocalStorage<NutrientRecipe[]>(RECIPES_KEY, []);
  const [appState, setAppState] = useState<AppState>({ ...INITIAL_APP_STATE, userRecipes: persistedRecipes, selectedBrands: [], excludedProducts: [] });

  const allAvailableRecipes = useMemo(() => [...PERMANENT_RECIPES, ...appState.userRecipes], [appState.userRecipes]);
  
  const computedResults = useMemo(() => compute(appState), [appState]);

  const { availableBrands, allProductNames } = useMemo(() => {
    const brands = new Set<string>();
    const productNames = Object.keys(FERTILIZER_DATABASE);
    productNames.forEach(name => {
      const brand = name.split(' ')[0];
      brands.add(brand);
    });
    return { availableBrands: Array.from(brands).sort(), allProductNames: productNames.sort() };
  }, []);
  
  // Persist recipes to localStorage whenever they change
  useEffect(() => {
    setPersistedRecipes(appState.userRecipes);
  }, [appState.userRecipes, setPersistedRecipes]);

  // When a recipe or week is selected, automatically set the products and their doses from the recipe.
  useEffect(() => {
    if (appState.selectedRecipeId && appState.selectedWeek !== null) {
        const recipe = allAvailableRecipes.find(r => r.id === appState.selectedRecipeId);
        const weekData = recipe?.weeks.find(w => w.week === appState.selectedWeek);

        if (weekData && weekData.doses && Object.keys(weekData.doses).length > 0) {
            const newProducts = Object.entries(weekData.doses)
                .map(([productName, dose]) => {
                    const profile = FERTILIZER_DATABASE[productName];
                    if (!profile) return null;
                    return {
                        id: `product-${productName}-${Date.now()}`,
                        name: productName,
                        dose_ml: dose,
                        profile: profile,
                    };
                })
                .filter((p): p is Product => p !== null);
            
            setAppState(prev => ({ ...prev, products: newProducts }));

        } else if (appState.products.some(p => p.name !== '')) {
             // If week has no doses, clear the product list
             setAppState(prev => ({
                ...prev,
                products: [{ ...DEFAULT_PRODUCT_ROW, id: `product-${Date.now()}` }]
            }));
        }
    } else if (!appState.selectedRecipeId) { // "Standard Phasen" selected
        setAppState(prev => ({
            ...prev,
            products: [
                { ...DEFAULT_PRODUCT_ROW, id: `product-${Date.now()}-0` },
                { ...DEFAULT_PRODUCT_ROW, id: `product-${Date.now()}-1` },
                { ...DEFAULT_PRODUCT_ROW, id: `product-${Date.now()}-2` },
            ]
        }));
    }
  }, [appState.selectedRecipeId, appState.selectedWeek, allAvailableRecipes]);


  const handleProductChange = useCallback((index: number, updatedProduct: Partial<Product>) => {
    setAppState(prevState => {
      const newProducts = [...prevState.products];
      newProducts[index] = { ...newProducts[index], ...updatedProduct };
      return { ...prevState, products: newProducts };
    });
  }, []);

  const handleAddProduct = useCallback(() => {
    setAppState(prevState => ({
      ...prevState,
      products: [...prevState.products, { ...DEFAULT_PRODUCT_ROW, id: `product-${Date.now()}` }],
    }));
  }, []);

  const handleRemoveProduct = useCallback((indexToRemove: number) => {
    const productToRemove = appState.products[indexToRemove];
    if (
      (productToRemove.name && productToRemove.name.trim() !== '') ||
      (productToRemove.dose_ml && productToRemove.dose_ml > 0)
    ) {
      if (!window.confirm(`Möchten Sie das Produkt "${productToRemove.name || 'leere Zeile'}" wirklich entfernen?`)) {
        return;
      }
    }
    setAppState(prevState => ({
      ...prevState,
      products: prevState.products.filter((_, index) => index !== indexToRemove),
    }));
  }, [appState.products]);

  const handlePhaseChange = useCallback((event: React.ChangeEvent<HTMLSelectElement>) => {
    setAppState(prevState => ({ ...prevState, phase: event.target.value as Phase }));
  }, []);

  const handleWaterShareChange = useCallback((event: React.ChangeEvent<HTMLInputElement>) => {
    const share = Math.max(0, Math.min(100, parseFloat(event.target.value) || 0));
    setAppState(prevState => ({
      ...prevState,
      waterInput: { ...prevState.waterInput, share_percent: share },
    }));
  }, []);
  
  const handleWaterProfileChange = useCallback((newProfile: DetailedWaterProfile) => {
      setAppState(prevState => ({ ...prevState, detailedWaterProfile: newProfile }));
    }, []);

  // Recipe Handlers
  const handleRecipeChange = useCallback((event: React.ChangeEvent<HTMLSelectElement>) => {
    const recipeId = event.target.value;
    setAppState(prevState => ({
      ...prevState,
      selectedRecipeId: recipeId === 'standard' ? null : recipeId,
      selectedWeek: null,
    }));
  }, []);
  
  const handleWeekChange = useCallback((event: React.ChangeEvent<HTMLSelectElement>) => {
    const week = event.target.value ? Number(event.target.value) : null;
    setAppState(prevState => ({ ...prevState, selectedWeek: week }));
  }, []);
  
  const handleSaveRecipe = (updatedRecipe: NutrientRecipe) => {
    setAppState(prevState => {
      const updatedRecipes = [...prevState.userRecipes];
      const existingIndex = updatedRecipes.findIndex(r => r.id === updatedRecipe.id);
      if (existingIndex > -1) {
        updatedRecipes[existingIndex] = updatedRecipe;
      } else {
        const newRecipeWithId = { ...updatedRecipe, id: String(Date.now()) };
        updatedRecipes.push(newRecipeWithId);
      }
      return { ...prevState, userRecipes: updatedRecipes, isRecipeEditorOpen: false, editingRecipe: null };
    });
  };
  
  const handleOpenRecipeEditor = useCallback((recipe: NutrientRecipe | null) => {
    setAppState(prevState => ({ ...prevState, isRecipeEditorOpen: true, editingRecipe: recipe }));
  }, []);
  
  const handleBrandChange = (selected: string[]) => {
      setAppState(prev => ({ ...prev, selectedBrands: selected }));
  }

  const handleExcludedProductsChange = (selected: string[]) => {
      setAppState(prev => ({ ...prev, excludedProducts: selected }));
  };
  
  const activeTargets = useMemo(() => {
    if (appState.selectedRecipeId) {
      const recipe = allAvailableRecipes.find(r => r.id === appState.selectedRecipeId);
      if (recipe && appState.selectedWeek !== null) {
        const week = recipe.weeks.find(w => w.week === appState.selectedWeek);
        if (week) return week.targets;
      }
    }
    return PHASE_TARGETS[appState.phase];
  }, [appState.phase, appState.selectedRecipeId, appState.selectedWeek, allAvailableRecipes]);

  const handleSuggestDoses = useCallback(() => {
    if (!activeTargets) return;
    
    const productsToConsider = Object.entries(FERTILIZER_DATABASE)
      .filter(([name]) => {
        // Filter by selected brands (if any)
        if (appState.selectedBrands.length > 0) {
          const brand = name.split(' ')[0];
          if (!appState.selectedBrands.includes(brand)) {
            return false;
          }
        }
        // Filter by excluded products
        if (appState.excludedProducts.includes(name)) {
          return false;
        }
        return true;
      })
      .map(([name, profile]) => ({ name, profile }));

    const suggestion = suggestDoses(activeTargets, productsToConsider, appState.detailedWaterProfile, appState.waterInput.share_percent);
    
    if (suggestion.length > 0) {
      setAppState(prev => ({ ...prev, products: suggestion }));
    } else {
      alert("Es konnte keine passende Düngerkombination für die ausgewählten Marken gefunden werden.");
    }
  }, [activeTargets, appState.selectedBrands, appState.excludedProducts, appState.detailedWaterProfile, appState.waterInput.share_percent]);

  const selectedRecipe = allAvailableRecipes.find(r => r.id === appState.selectedRecipeId);
  const canSuggestDoses = !!appState.selectedRecipeId && appState.selectedWeek !== null;
  
  return (
    <div className="min-h-screen bg-gray-900 text-gray-200 p-4 sm:p-6 lg:p-8">
      <div className="max-w-7xl mx-auto">
        <header className="mb-8">
          <h1 className="text-4xl font-bold text-green-400">Nährstoff-Rechner</h1>
          <p className="text-gray-400 mt-1">Planen und optimieren Sie Ihre Nährlösung.</p>
        </header>

        <div className="grid grid-cols-1 lg:grid-cols-3 gap-8">
          <main className="lg:col-span-2 space-y-6">
            {/* Settings Section */}
            <section className="bg-gray-800 p-5 rounded-xl">
                <h2 className="text-2xl font-semibold mb-4 text-gray-200">Einstellungen</h2>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6">
                    <div>
                        <label className="block text-sm font-medium text-gray-400 mb-2">Schema</label>
                        <SelectInput value={appState.selectedRecipeId || 'standard'} onChange={handleRecipeChange}>
                          <option value="standard">Standard Phasen</option>
                          {allAvailableRecipes.map(r => <option key={r.id} value={r.id}>{r.name}</option>)}
                        </SelectInput>
                    </div>
                     <div>
                        <label className="block text-sm font-medium text-gray-400 mb-2">Woche / Phase</label>
                        {appState.selectedRecipeId ? (
                            <SelectInput value={appState.selectedWeek?.toString() || ''} onChange={handleWeekChange} disabled={!selectedRecipe}>
                                <option value="">Woche auswählen</option>
                                {selectedRecipe?.weeks.sort((a,b) => a.week - b.week).map(w => <option key={w.week} value={w.week}>{w.name || `Woche ${w.week}`}</option>)}
                            </SelectInput>
                        ) : (
                             <SelectInput value={appState.phase} onChange={handlePhaseChange}>
                                {Object.values(Phase).map(p => <option key={p} value={p}>{p}</option>)}
                            </SelectInput>
                        )}
                    </div>
                </div>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mt-4">
                     <div>
                        <label className="block text-sm font-medium text-gray-400 mb-2">Bevorzugte Marken</label>
                        <BrandSelector brands={availableBrands} selectedBrands={appState.selectedBrands} onChange={handleBrandChange} />
                    </div>
                     <div>
                        <label className="block text-sm font-medium text-gray-400 mb-2">Produkte ausschließen</label>
                        <ProductMultiSelector allProducts={allProductNames} selectedProducts={appState.excludedProducts} onChange={handleExcludedProductsChange} />
                    </div>
                </div>
                <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mt-4">
                    <div>
                        <button 
                            onClick={handleSuggestDoses}
                            disabled={!canSuggestDoses}
                            className="w-full h-full py-2 px-4 bg-indigo-600 hover:bg-indigo-700 rounded-md transition font-semibold disabled:bg-gray-600 disabled:opacity-50 disabled:cursor-not-allowed"
                        >
                            Düngung vorschlagen
                        </button>
                    </div>
                     <div className="flex items-end flex-wrap gap-2">
                        <button onClick={() => handleOpenRecipeEditor(null)} className="py-2 px-4 bg-green-600 hover:bg-green-700 rounded-md transition text-sm font-semibold">Neues Schema</button>
                         {selectedRecipe && !selectedRecipe.isPermanent && <button onClick={() => handleOpenRecipeEditor(selectedRecipe)} className="py-2 px-4 bg-gray-600 hover:bg-gray-700 rounded-md transition text-sm font-semibold">Schema bearbeiten</button>}
                    </div>
                </div>
            </section>
          
            {/* Products Section */}
            <section className="bg-gray-800 p-5 rounded-xl">
              <h2 className="text-2xl font-semibold mb-4 text-gray-200">Produkte</h2>
              <div className="space-y-3">
                {appState.products.map((p, index) => (
                  <ProductRow
                    key={p.id}
                    index={index}
                    product={p}
                    onProductChange={handleProductChange}
                    onRemoveProduct={handleRemoveProduct}
                    fertilizerDatabase={FERTILIZER_DATABASE}
                  />
                ))}
              </div>
              <button onClick={handleAddProduct} className="mt-4 w-full py-2 px-4 bg-green-800/50 hover:bg-green-700/50 border border-dashed border-green-600 rounded-md transition font-semibold text-green-300">
                + Produkt hinzufügen
              </button>
            </section>

            {/* Water Section */}
             <section className="bg-gray-800 p-5 rounded-xl">
                <h2 className="text-2xl font-semibold mb-4 text-gray-200">Wasserprofil</h2>
                {computedResults.ecWarning && (
                    <div className={`p-3 mb-4 border rounded-md text-sm ${computedResults.ecWarning.className}`}>
                        {computedResults.ecWarning.message}
                    </div>
                )}
                <div className="flex flex-col sm:flex-row gap-4 items-center">
                    <div className="flex-grow w-full">
                        <label className="block text-sm font-medium text-gray-400 mb-2">Anteil Leitungswasser (%)</label>
                        <NumberInput value={appState.waterInput.share_percent} onChange={handleWaterShareChange} />
                    </div>
                    <button onClick={() => setAppState(p => ({...p, isWaterAnalysisDialogOpen: true}))} className="w-full sm:w-auto mt-2 sm:mt-7 py-2 px-5 bg-blue-600 hover:bg-blue-700 rounded-md transition font-semibold">
                        Wasseranalyse anpassen
                    </button>
                </div>
            </section>
          </main>

          <aside>
            <NutrientSumDisplay computedResults={computedResults} activeTargets={activeTargets} />
          </aside>
        </div>
      </div>
      
      <WaterAnalysisDialog 
        isOpen={appState.isWaterAnalysisDialogOpen}
        onClose={() => setAppState(p => ({...p, isWaterAnalysisDialogOpen: false}))}
        onSave={handleWaterProfileChange}
        initialProfile={appState.detailedWaterProfile}
      />
      <RecipeEditorDialog
        isOpen={appState.isRecipeEditorOpen}
        onClose={() => setAppState(p => ({...p, isRecipeEditorOpen: false, editingRecipe: null}))}
        onSave={handleSaveRecipe}
        recipeToEdit={appState.editingRecipe}
      />
    </div>
  );
};

export default App;
