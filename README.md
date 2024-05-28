# IOFC function templates
Dave Innes

*This document is a rendered version of `IOFC_template.ipynb` (via Quarto).*

Income over feed cost (IOFC) is calculated below using 2 methods:

1)  manually entered values, typical of a whole-herd IOFC calculation
    (also see `IOFC calculator template.xlsx`)
2)  a function that uses the output from the `nasem_dairy` package which
    is useful for calculating IOFC from predicted values

## Define revenue and ingredient prices

Milk prices (\$/kg) are based on a typical on-farm milk cheque, ignoring
the bulk correction for SNF (solids not fat) to Fat ratio which isn’t
possible for individual cows. The `milk_prices` dict also has
`milk_other_solids_production_perc` which is typically fixed and not a
value calculated by the model. The value of this component of the milk
is `other_solids_dollar_kg`. The % of fat and protein are taken from the
ModelOutput object returned by `nd.execute_model()`.

Ingredient prices are \$/as-fed tonne (i.e. not on DM basis).

``` python
import pandas as pd
```

``` python
milk_prices = {
    'milk_other_solids_production_perc' : 5.7, #mostly lactose (4.7%) and minerals (1%)
    'milk_fat_dollar_kg' : 13.3,
    'milk_protein_dollar_kg' : 10.96,
    'other_solids_dollar_kg' : 0.9,
    
}


# $/tonne
ingredient_prices_asfed_tonne = {
    'Wheat straw, Elora': 200,  
    'Alfalfa Silage': 230,  
    'Corn silage, Elora': 180,  
    'Corn Grain HM': 250,  
    'Soy Plus': 700,  
    'Soybean meal 47%': 600, 
    'Canola': 600,  
    'Wheat Shorts': 250,  
    'Tallow': 500,  
    'Sodium chloride (salt)': 1000,  
    'Limestone calcium carbonate': 1000,  
    'Monocalcium phosphate(Dical)': 1000,  
    'Magnesium oxide, Elora': 1000,  
    'Diamond V yeast': 2000,  
    'DCAD+ (Potassium carbonate)': 1000,  
    'Metasmart': 2000,
    'Vitamin/mineral mix': 2000
}
```

## 1. Manual IOFC

This code does the same as `IOFC calculator template.xlsx`.

This also requires an additional dictionary to define the milk
components and ration, as the model output is not used in this case.

``` python
# Note that the 'milk_other_solids_production_perc' is already in milk_prices
milk_components = {
   'MY_kg_d' : 31,
   'Milk_Fat_perc' : 3.6,
   'Milk_TProt_perc' : 3.3,
}


# Define the data as a list of dictionaries
ration_list = [
    {"Ingredient": "Wheat straw, Elora",   "kg_inclusion": 0.5,  "DM_perc": 90},
    {"Ingredient": "Alfalfa Silage",       "kg_inclusion": 8.2,  "DM_perc": 35},
    {"Ingredient": "Corn silage, Elora",   "kg_inclusion": 8.2,  "DM_perc": 33},
    {"Ingredient": "Corn Grain HM",        "kg_inclusion": 3.7,  "DM_perc": 87},
    {"Ingredient": "Soybean meal 47%",     "kg_inclusion": 2.0,  "DM_perc": 88},
    {"Ingredient": "Canola",               "kg_inclusion": 1.0,  "DM_perc": 91},
    {"Ingredient": "Vitamin/mineral mix",  "kg_inclusion": 0.2,  "DM_perc": 100}
]

# Convert to a DataFrame
ration_df = pd.DataFrame(ration_list)
```

### Create function

``` python
def calc_IOFC_manual(
        milk_prices: dict,
        milk_components: dict,
        ingredient_prices: dict, # must contain matches to all feedstuffs in ration
        ration_df: pd.DataFrame,
        verbose: bool = False
        ) -> float:
    """
    Calculate Income Over Feed Cost (IOFC)

    The basic calculation is `milk revenue - feed cost` per cow per day. 
    Milk revenue is based on price paid for milk fat, true protein and other 
    solids. Each of these components is provided to the function as a %, and are
    multiplied by the provided milk yield (kg/d) to get component yields (kg/d).
    A ration is also provided as a dataframe with kg_inclusion (kg/d)
    and DM_perc (dry matter %) for each ingredient. These are required so that 
    the ingredient prices (as-fed) can be calculated on a DM basis.


    Parameters
    ----------
    milk_prices : dict
        prices in $/kg. Also includes `milk_other_solids_production_perc` which 
        is typically fixed 5.7% (4.7% lactose and 1% minerals)
    
    milk_components: dict
        A dictionary with milk yield (kg/d), milk fat (%), and true protein (%)
    
    ingredient_prices : dict
        prices in $/tonne. Any ingredients can be included in this dict but only
         those that match an ingredient in `ration_df` will be used.

    ration_df: pd.DataFrame
        A dataframe with 3 columns: Ingredient, kg_inclusion, DM_perc which
        represents the diet being fed on a DM basis. The sum of kg_inclusion should
        match their expected DM intake. DM_perc is used to adjust ingredient_prices.

    verbose: bool
        If True - intermediate calculations are printed to console. Default is False.


    Returns
    -------
    float
        A single value for IOFC in $/cow/day
    """    
    ###########################
    # Input validation
    ###########################
    # Check ration_df ingredients have matching price
    ingredient_validation = ration_df['Ingredient'].apply(lambda x: x in ingredient_prices_asfed_tonne)

    if not ingredient_validation.all():
        unmatched_ingredients = ration_df[~ingredient_validation]['Ingredient'].tolist()
        raise ValueError(f"The following ingredients do not have a match in the price list: {unmatched_ingredients}")

    
    ###########################
    # Component Production kg/d
    ###########################
    MY_kg_d = milk_components['MY_kg_d']

    milk_fat_kg_d = milk_components['Milk_Fat_perc']/100 * MY_kg_d
    milk_TP_kg_d = milk_components['Milk_TProt_perc']/100 * MY_kg_d
    milk_other_kg_d = milk_prices['milk_other_solids_production_perc']/100 * MY_kg_d 
  
    if verbose:
        print(f"Milk Fat kg/d = {round(milk_fat_kg_d, 2)}, "
              f"True Protein kg/d = {round(milk_TP_kg_d, 2)}, "
              f"Other Solids kg/d = {round(milk_other_kg_d, 2)}.\n")

    ###########################
    # Revenue $/d
    ###########################
    milk_fat_dollar_d = milk_fat_kg_d * milk_prices['milk_fat_dollar_kg']
    milk_protein_dollar_d = milk_TP_kg_d * milk_prices['milk_protein_dollar_kg']
    milk_other_dollar_d = milk_other_kg_d * milk_prices['other_solids_dollar_kg']

    revenue_dollar_per_d = (
        milk_fat_dollar_d + 
        milk_protein_dollar_d + 
        milk_other_dollar_d)

    if verbose:
        print(f"Milk Fat dollar/d = {round(milk_fat_dollar_d,2)}, "
              f"True Protein dollar/d = {round(milk_protein_dollar_d,2)}, " 
              f"Other Solids dollar/d = {round(milk_other_dollar_d,2)}\n")
        print(f"Revenue $/d = {round(revenue_dollar_per_d,2)}\n")

    ###########################
    # Feed Costs
    ###########################
    
    # calculate daily costs ($/d) for each ingredient
    ration_df = ration_df.assign(
        price_asfed_t = lambda df: df['Ingredient'].map(ingredient_prices),
        price_DM_t = lambda df: df['price_asfed_t'] / df['DM_perc'] * 100,
        price_DM_kg = lambda df: df['price_DM_t'] / 1000,
        daily_cost = lambda df: df['kg_inclusion'] * df['price_DM_kg']
    )
    
    diet_cost_p_d = ration_df['daily_cost'].sum()

    if verbose:
        print(ration_df)
        print(f"\nDiet cost ($/d) = {round(diet_cost_p_d,2)}\n")

    ###########################
    # IOFC
    ###########################
    IOFC = revenue_dollar_per_d - diet_cost_p_d

    if verbose:
        print(f"IOFC ($/d) = {round(IOFC,2)}")

    return IOFC
```

### Execute function

Run model with verbose = True will print out all intermediate
calculations.

``` python
calc_IOFC_manual(
        milk_prices = milk_prices,
        milk_components = milk_components,
        ingredient_prices = ingredient_prices_asfed_tonne,
        ration_df = ration_df,
        verbose = True
        )
```

    Milk Fat kg/d = 1.12, True Protein kg/d = 1.02, Other Solids kg/d = 1.77.

    Milk Fat dollar/d = 14.84, True Protein dollar/d = 11.21, Other Solids dollar/d = 1.59

    Revenue $/d = 27.65

                Ingredient  kg_inclusion  DM_perc  price_asfed_t   price_DM_t  \
    0   Wheat straw, Elora           0.5       90            200   222.222222   
    1       Alfalfa Silage           8.2       35            230   657.142857   
    2   Corn silage, Elora           8.2       33            180   545.454545   
    3        Corn Grain HM           3.7       87            250   287.356322   
    4     Soybean meal 47%           2.0       88            600   681.818182   
    5               Canola           1.0       91            600   659.340659   
    6  Vitamin/mineral mix           0.2      100           2000  2000.000000   

       price_DM_kg  daily_cost  
    0     0.222222    0.111111  
    1     0.657143    5.388571  
    2     0.545455    4.472727  
    3     0.287356    1.063218  
    4     0.681818    1.363636  
    5     0.659341    0.659341  
    6     2.000000    0.400000  

    Diet cost ($/d) = 13.46

    IOFC ($/d) = 14.19

    14.186574773808571

``` python
# verbose = False
calc_IOFC_manual(
        milk_prices = milk_prices,
        milk_components = milk_components,
        ingredient_prices = ingredient_prices_asfed_tonne,
        ration_df = ration_df,
        verbose = False
        )
```

    14.186574773808571

``` python
ration_df['kg_inclusion'].sum()
```

    23.799999999999997

## 2. IOFC using nasem_dairy (nd) package

In this function the fat and protein % are taken from the ModelOutput
object returned by `nd.execute_model()`.

``` python
import nasem_dairy as nd
import pandas as pd
```

### Define function

``` python

def calc_IOFC_nd(
        milk_prices: dict,
        ingredient_prices: dict, # must contain matches to all feedstuffs in user_diet
        mod_output: nd.ModelOutput
        ) -> float:
    """
    Calculate Income Over Feed Cost (IOFC) from nasem_dairy output.

    The basic calculation is `milk revenue - feed cost` per cow per day. 
    Milk revenue is based on milk component pricing, with `Mlk_Fat_g` and 
    `Mlk_NP_g` returned from `nasem_dairy` package (i.e `nd`) representing milk 
    fat and protein, respectively. These could be 'target' values that are entered
    by the user, or they could be 'predicted' values based on the model. The user
    must decide to use either target or prediction equations when executing `nd`.

    Parameters
    ----------
    milk_prices : dict
        prices in $/kg. Also includes `milk_other_solids_production_perc` which 
        is typically fixed 5.7% (4.7% lactose and 1% minerals)
    ingredient_prices : dict
        prices in $/tonne. Ingredients in this dict should match all ingredients 
        used in the `user_diet` when running `nd.execute_model()` to get `mod_output`.
    mod_output: nd.ModelOutput
        An object that is returned by `nd.execute_model()`

    Returns
    -------
    float
        A single value for IOFC in $/cow/day
    """    
    
  
    ##########
    # Revenue
    ##########
    milk_fat_price_p_d = (mod_output.get_value('Mlk_Fat_g')/1000 * 
                          milk_prices['milk_fat_dollar_kg'])
    
    # assume it is True Protein (not Crude Protein)
    milk_protein_price_p_d = (mod_output.get_value('Mlk_NP_g')/1000 * 
                              milk_prices['milk_protein_dollar_kg'])

    # calculated from a fixed % of milk yield
    MY_kg_d = mod_output.get_value('Mlk_Prod')

    milk_other_solids_price_p_d = (
        milk_prices['milk_other_solids_production_perc']/100 *
        MY_kg_d * 
        milk_prices['other_solids_dollar_kg'])

    revenue_dollar_per_d = (
        milk_fat_price_p_d + 
        milk_protein_price_p_d + 
        milk_other_solids_price_p_d)


    #############
    # Feed Costs
    #############

    # get diet info from output: 
    diet_sub = (mod_output
                .Intakes['diet_info']
                .filter(['Feedstuff', 'Fd_DMInp', 'Fd_DM', 'Fd_DMIn'])
                .assign(
                    dollar_t_as_fed = lambda df: df['Feedstuff'].map(ingredient_prices),
                    dollar_t_DM = lambda df: df['dollar_t_as_fed'] / df['Fd_DM']*100,
                    dollar_fed_p_d = lambda df: df['dollar_t_DM']/1000 * df['Fd_DMIn'])
                )
    
    diet_cost_p_d = diet_sub['dollar_fed_p_d'].sum()


    #############
    # IOFC
    #############
    IOFC = revenue_dollar_per_d - diet_cost_p_d

    return IOFC

```

### Prepare data for nasem_dairy

``` python
# Diet
diet_info = pd.DataFrame([
    {"Feedstuff": "Wheat straw",   "kg_user": 0.5},
    {"Feedstuff": "Alfalfa Silage",       "kg_user": 8.2},
    {"Feedstuff": "Corn silage",   "kg_user": 8.2},
    {"Feedstuff": "Corn Grain HM",        "kg_user": 3.7},
    {"Feedstuff": "Soybean meal 47%",     "kg_user": 2.0},
    {"Feedstuff": "Canola",               "kg_user": 1.0},
    {"Feedstuff": "Vitamin/mineral mix",  "kg_user": 0.2}
])

# Animal Inputs
animal_inputs = {
    'An_Parity_rl': 1.33,
    'Trg_MilkProd': 25.062,
    'An_BW': 624.795,
    'An_BCS': 3.2,
    'An_LactDay': 170.0,
    'Trg_MilkFatp': 4.55,
    'Trg_MilkTPp': 3.66,
    'Trg_MilkLacp': 4.85,
    'DMI': 24.0,
    'An_BW_mature': 700.0,
    'Trg_FrmGain': 0,
    'An_GestDay': 46.0,
    'An_GestLength': 280.0,
    'Trg_RsrvGain': 0.0,
    'Fet_BWbrth': 44.1,
    'An_AgeDay': 820.8,
    'An_305RHA_MlkTP': 280.0,
    'An_StatePhys': 'Lactating Cow',
    'An_Breed': 'Holstein',
    'An_AgeDryFdStart': 14.0,
    'Env_TempCurr': 22.0,
    'Env_DistParlor': 0.0,
    'Env_TripsParlor': 0.0,
    'Env_Topo': 0.0}

# Model equations - these are currently 'predictive' eqns
equation_selection = {
            'Use_DNDF_IV': 0.0,
            'DMIn_eqn': 8, # 8 = predicted, 0 = Target (9 should be an option too when package fixed)
            'mProd_eqn': 1, # 1 = predicted, 0 = Target
            'MiN_eqn': 1.0,
            'use_infusions': 0.0,
            'NonMilkCP_ClfLiq': 0.0,
            'Monensin_eqn': 0.0,
            'mPrt_eqn': 0.0, # always predicted - until updated
            'mFat_eqn': 1,  # 1 = predicted, 0 = Target
            'RumDevDisc_Clf': 0.0}



```

Import feed library (with missing column; see [Issue
\#82](https://github.com/CNM-University-of-Guelph-dev/NASEM-Model-Python/issues/82))

``` python
feed_library_in = pd.read_csv( "./template_feed_library.csv").assign(Fd_DNDF48 = 0.0)
```

### Execute `nd.execute_model()`

There may still be some warnings appear, but output will be stored in
`NASEM_output`

``` python
NASEM_output = nd.execute_model(
    user_diet = diet_info, 
    animal_input = animal_inputs, 
    equation_selection = equation_selection, 
    feed_library_df = feed_library_in) 

NASEM_output
```

        <div>
            <h2>Model Output Snapshot</h2>
            &#10;
| Description                                         | Value    |
|-----------------------------------------------------|----------|
| Milk production kg (Mlk_Prod_comp)                  | 29.741   |
| Milk fat g/g (MlkFat_Milk)                          | 0.036    |
| Milk protein g/g (MlkNP_Milk)                       | 0.033    |
| Milk Production - MP allowable kg (Mlk_Prod_MPalow) | 26.635   |
| Milk Production - NE allowable kg (Mlk_Prod_NEalow) | 32.793   |
| Animal ME intake Mcal/d (An_MEIn)                   | 60.149   |
| Target ME use Mcal/d (Trg_MEuse)                    | 50.446   |
| Animal MP intake g/d (An_MPIn_g)                    | 1994.160 |
| Animal MP use g/d (An_MPuse_g_Trg)                  | 1910.743 |
| Animal RDP intake g/d (An_RDPIn_g)                  | 2381.933 |
| Diet DCAD meq (An_DCADmeq)                          | 221.438  |

            <hr>
            &#10;        <details>
            <summary><strong>Click this drop-down for ModelOutput description</strong></summary>
            <p>This is a <code>ModelOutput</code> object returned by <code>nd.execute_model()</code>.</p>
            <p>Each of the following categories can be called directly as methods, for example, if the name of my object is <code>output</code>, I would call <code>output.Production</code> to see the contents of Production.</p>
            <p>The following list shows which dictionaries are within each category:</p>
            <ul>
        <li><b>Inputs:</b> user_diet, animal_input, equation_selection, coeff_dict, infusion_input, MP_NP_efficiency_input, mPrt_coeff, f_Imb</li><li><b>Intakes:</b> diet_info, infusion_data, diet_data, An_data, energy, protein, AA, FA, rumen_digestable, water</li><li><b>Requirements:</b> energy, protein, vitamin, mineral, mineral_requirements</li><li><b>Production:</b> milk, body_composition, gestation, MiCP</li><li><b>Excretion:</b> fecal, urinary, gaseous, scurf</li><li><b>Digestibility:</b> rumen, TT</li><li><b>Efficiencies:</b> energy, protein, mineral</li><li><b>Miscellaneous:</b> misc</li>
            </ul>
            <div>
                <p>These outputs can be accessed by name, e.g., <code>output.Production['milk']['Mlk_Prod']</code>.</p>
                <p>There is also a <code>.search()</code> method which takes a string and will return a dataframe of all outputs with that string (case insensitive), e.g., <code>output.search('Mlk')</code>.</p>
                <p>An individual output can be retrieved directly by providing its exact name to the <code>.get_value()</code> method, e.g., <code>output.get_value('Mlk_Prod')</code>.</p>
            </div>
        </details>
        &#10;        </div>
        &#10;
### Calculate IOFC

This uses the prices defined at top of this file and the `NASEM_output`
object calculated above

``` python
IOFC = calc_IOFC_nd(milk_prices, ingredient_prices_asfed_tonne, NASEM_output)

# print IOFC in $/d
IOFC
```

    19.119754582521328
