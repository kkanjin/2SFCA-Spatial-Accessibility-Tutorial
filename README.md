# 2SFCA-Spatial-Accessibility-Tutorial
Calculating Spatial Accessibility Using 2SFCA

A step-by-step tutorial for computing spatial accessibility with the Two-Step Floating Catchment Area (2SFCA) method in ArcGIS Pro.

Author: Kingsley Kanjin Worked example: Spatial accessibility to pediatric dentists in Washington, DC

About this tutorial

This walkthrough adapts the 2SFCA procedure described in Wang & Liu (2023), Computational Methods and GIS Applications in Social Science, Chapter 5. The original procedure has been applied here to a new case study — access to pediatric dental care across Washington, DC census tracts.

Written for ArcGIS Pro 3.2, though the workflow translates to ArcMap or any GIS package with equivalent geoprocessing tools.

What 2SFCA measures

2SFCA produces an accessibility score for every population unit that accounts for both supply (how many providers are reachable) and demand (how many other people are competing for those same providers). A tract with three clinics nearby but 50,000 neighbours may score lower than a tract with one clinic and few competitors.

The method runs in two passes:

Supply-to-demand ratio at each facility — for every provider, sum the population within the catchment distance and divide provider capacity by that population.
Accessibility at each population location — for every tract, sum the ratios of all providers within the catchment distance.
𝐴
𝑖
=
∑
𝑗
∈
{
𝑑
𝑖
𝑗
≤
𝑑
0
}
𝑅
𝑗
=
∑
𝑗
∈
{
𝑑
𝑖
𝑗
≤
𝑑
0
}
(
𝑆
𝑗
∑
𝑘
∈
{
𝑑
𝑘
𝑗
≤
𝑑
0
}
𝐷
𝑘
)
A
i
	​

=
j∈{d
ij
	​

≤d
0
	​

}
∑
	​

R
j
	​

=
j∈{d
ij
	​

≤d
0
	​

}
∑
	​

(
∑
k∈{d
kj
	​

≤d
0
	​

}
	​

D
k
	​

S
j
	​

	​

)

Where:

Term	Meaning

𝐴
𝑖
A
i
	​

	Accessibility score at population location i

𝑅
𝑗
R
j
	​

	Supply-to-demand ratio at facility j

𝑆
𝑗
S
j
	​

	Supply capacity at facility j (e.g. number of dentists)

𝐷
𝑘
D
k
	​

	Demand (population) at location k

𝑑
𝑖
𝑗
d
ij
	​

	Distance between population i and facility j

𝑑
0
d
0
	​

	Catchment threshold distance
Data requirements

Both datasets must be in the same projected coordinate system before you begin.

Population units — census tracts, block groups, or community polygons. Must contain a population or household field. Example used here: Census_tract_2020_proj

Facility locations — point features representing providers. Include a capacity field (number of staff) if you are measuring access to service; omit it if you only need access to facility count. Example used here: PediatricDentists_prj

Step 1 — Compute distances

Geoprocessing → Generate Near Table

Parameter	Value
Input Features	Census_tract_2020_proj
Near Features	PediatricDentists_prj
Output Table	Distance_All_Tab
Method	Planar
Maximum number of closest matches	0 (returns all pairs)

Setting the maximum matches to 0 is important — 2SFCA needs every population-to-facility pair within the catchment, not just the nearest one.

Alternative: an OD cost matrix will give network distances rather than Euclidean, which is more realistic for travel. You will need to generate polygon centroids first.

Step 2 — Join population and facility attributes

Join both source tables to the distance table:

Join	Source field	Target field
Census_tract_2020_proj → Distance_All_Tab	FID	IN_FID
PediatricDentists_prj → Distance_All_Tab	OBJECTID	NEAR_FID
Step 3 — Extract distances within the catchment

Select By Attributes on Distance_All_Tab:

NEAR_DIST <= 8047

8047 metres ≈ 5 miles. Export the selection to a new table, Distance5mi.

This implements the two selection conditions in the equation: 
𝑑
𝑖
𝑗
≤
𝑑
0
d
ij
	​

≤d
0
	​

 and 
𝑑
𝑘
𝑗
≤
𝑑
0
d
kj
	​

≤d
0
	​

.

Choosing 
𝑑
0
d
0
	​

: the catchment distance should reflect realistic travel behaviour for the service being studied and ideally follow an established standard in the literature. It is the single most influential parameter in the model — document your justification.

Step 4 — Sum population around each facility

Summary Statistics on Distance5mi:

Parameter	Value
Statistics Field	population field (e.g. P001001)
Statistic Type	Sum
Case Field	NEAR_FID
Output Table	Dentists5mi

The resulting SUM_P001001 field is the total population within the catchment of each facility — the denominator 
∑
𝑘
𝐷
𝑘
∑
k
	​

D
k
	​

.

Step 5 — Compute the supply-to-demand ratio

Join Dentists5mi back to Distance5mi on NEAR_FID.

Add a field Dentists (Long) and populate it with the capacity at each facility. If actual staff counts are unavailable, assign a uniform value and state the assumption.

Add a field DentPopR (Double) and calculate:

python
DentPopR = 1000 * !Distance5mi.Dentists! / !Dentists5mi.SUM_P001001!

This computes 
𝑆
𝑗
/
∑
𝑘
𝐷
𝑘
S
j
	​

/∑
k
	​

D
k
	​

. The 1,000 multiplier expresses the result as providers per 1,000 residents — adjust to suit the density of your study area.

Step 6 — Sum ratios by population location

Remove the joins from Distance5mi, then run Summary Statistics again:

Parameter	Value
Statistics Field	DentPopR
Statistic Type	Sum
Case Field	IN_FID
Output Table	DentAccess5mi

SUM_DentPopR is the accessibility score 
𝐴
𝑖
A
i
	​

 for each population location.

Step 7 — Map the result

Geoprocessing → Join Field (a permanent join, so scores persist):

Parameter	Value
Input Table	Census_tract_2020_proj
Input Join Field	FID
Join Table	DentAccess5mi
Join Table Field	IN_FID
Transfer Fields	SUM_DentPopR

Symbolise the tracts by SUM_DentPopR. High values indicate high accessibility; low values indicate low accessibility.

Using the scripted tools

Wang & Liu (2023) provide the whole procedure as ArcGIS script tools, which avoids the manual table work above. Add Accessibility.tbx to your ArcGIS toolboxes:

Tool	Use
Two-Step Floating Catchment Area (2SFCA)	Standard 2SFCA, computes distances internally
Two-Step Floating Catchment Area (w External Distance Table)	Use when you already have a distance table
Generalized 2SFCA	Adds a distance decay function
Generalized 2SFCA (w External Distance Table)	As above, with existing distances

Adjusting the population multiplier: right-click the script tool → Edit, and locate the calculation block. The 1000.0 factor appears in each branch of the distance decay conditional:

python
# Calculate supply-to-demand ratio field with distance decay
if distanceDecayFunc == "Power":
    expression2 = "(1000.0 * !" + supplyFieldDoc + "! / !SUM_PPotent!) * !Weights!"
elif distanceDecayFunc == "Exponential":
    expression2 = "(1000.0 * !" + supplyFieldDoc + "! / !SUM_PPotent!) * math.exp(...)"
else:
    expression2 = "(1000.0 * !" + supplyFieldDoc + "! / !SUM_PPotent!) / (math.sqrt(2 * math.pi) * ...)"
Notes and caveats

Euclidean vs. network distance. Straight-line distance ignores road networks, rivers, and barriers. Network distance or travel time gives more defensible results, at the cost of longer processing and a routable network dataset.

The catchment threshold drives the outcome. A 5-mile catchment and a 20-minute drive-time catchment can produce quite different accessibility patterns from the same inputs. Test sensitivity to this parameter before drawing conclusions.

Basic 2SFCA treats everything inside the catchment equally. A facility 0.5 miles away counts the same as one 4.9 miles away. Generalized 2SFCA addresses this with a distance decay function and is usually the better choice for real analysis.

Edge effects. Facilities just outside the study area boundary are excluded, which understates accessibility near the edges. Consider buffering your facility dataset beyond the study boundary.

Reference

Wang, F., & Liu, L. (2023). Computational Methods and GIS Applications in Social Science (3rd ed.). CRC Press. Chapter 5.

Related work

This method underpins my published research on spatial accessibility to potable water in Sibi, Ghana — Applied Geography.
