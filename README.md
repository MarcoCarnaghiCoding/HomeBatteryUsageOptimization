# Home Battery Usage Optimization

The aim of the project is to develope an optimization problem to reduce energy costs associated with a building while maintaining appropriate operating conditions for a battery storage system.

## FIRST Attempt ITEM
The first step is to create a simple model for a battery storage system with 100% efficiency.

I decided to design the battery's capacity to cover the energy difference during periods of peak power load compared to the average load.

While I didn't have specific information about the technology of the battery, I adopted typical values for the maximum and minimum state of charge.

It's important to note that these values will vary depending on the type of battery used, such as deep-cycle lead acid, lithium-ion, lithium iron phosphate, or nickel-iron batteries.

In particular, these values have a great impact on the life of the battery in terms of the number of cycles of charging and discharging.


To track the state of charge of the battery, I defined a new CP variable and added additional methods to define associated constraints.

 These constraints were simply inequality constraints, similar to those related to the maximum charging and discharging rate.



I also defined a set of equality constraints that corresponded to the coherence or time evolution of the values in the state of charge vector

In this case, the delta in the state of charge variable should correspond to the aggregated power or energy charged or discharged from the battery.

## SECOND Atempt

### DEFINE COST FUNCTION
The cost function of the optimization problem returns the cost (positive) or revenue (negative) of the site based on a time-varying cost vector. The optimization objective is to minimize the cost while ensuring appropriate operating conditions for the battery storage system.

The cost vector is a numpy array where each element represents the price for a particular interval in $/kWh. 
The site can earn revenue when it exports to the grid (i.e., the aggregate power draw is negative) or pay for electricity when it imports from the grid. 
The price/revenue is the same for importing and exporting.
To calculate the energy cost at each time interval, I multiplied the time-varying price vector by the sum of the building load and battery power output/input, and the control interval time. The building load and battery power are summed because they represent the total power consumption or generation at each time interval.

It's important to note that the power units need to be converted from kW to kWh by multiplying by the control interval time (in hours) because the time-varying price vector is defined in units of $/kWh.

#### NOTE:
1. If charging and discharging losses should be considered and can be considered equal, this could be incorporated as a constant coefficient (k<1) in the equality constraint related to the state of charge update.

2. If the losses are considerably different, the problem becomes more complicated. In this case, it is necessary to model that losses are linear but with different slopes according to the power flow being from or to the battery.
This situation can be represented with a **Big-M** formulation and introducing a Boolean vector variable.
The idea would be to activate a certain set of constraints if the battery is charging and another set if the battery is discharging.

3. Another possible source of non linearity is the energy cost. If the price of the imported and exported energy is not equal, the cost function will need a similar re formulation as the mentioned for the battery losses.

4. Lastly, to plot the state of charge of the battery and ensure that the constraints were working properly, I modified the plotting method accordingly.


## THIRD Attempt
The third step is to Extend the model for a hypothetical battery which considers overheating:
1. When the battery charges or discharges the battery's temperature rises proportionally to the aggregated power / energy [ kWh].
2. The battery has a passive cooling system which is capable of reducing the battery's temperature by 1.0 deg C / minute.
3. The cooling system cannot reduce the battery to a temperature below the ambient air temperature.
4. For simplicity, I will assume the cooling rate is independent of the battery and ambient temperatures.

Then, I need to add constraint(s) so that the battery will not exceed 200 deg C at any time.

In this case, I considered three possible formulations to include the temperature constraints to the problem.
This formulation uses the following:
1. A strict restriction regarding the evolution of the Battery temperature.
Use of the Big M formulation to deal with the absolute value nature of the temperature rise made by power flow.
In this sense, to model the condition that:
```math
\begin{align*}
DELTA T_P 	&= |Power io|\; x\; Coefficient \;x \;control interval \\
		&=  |Power io \;x\; Coefficient\; x\; control interval| \\
		&== DELTA T_C + DELTA T_B
\end{align*}
```
AS both the coefficient and the control interval are positive. I wrote the condition as the interception of two conditions, i.e:
```latex
\begin{equation*}
DELTA T_P \geq K and DELTA T_P \leq KB
\end{equation*}
```
which have formulations declared in the literature.

In this case: 
```latex
\begin{equation*}
K = DELTA T_C + DELTA T_B
\end{equation*}
```
The formulation also considers that the temperature of the battery will always be above the ambient temperature.

**Explanation**: Since the cooling system cannot take the battery temperature below the ambient temperature, it is not physically possible for the battery temperature to be lower than the ambient temperature.
Therefore, it makes sense to constrain the battery temperature to be greater than or equal to the ambient temperature. This avoids the need to model the cooling system's behavior for temperatures below the ambient temperature and simplifies the overall model.

This Formulation is simple, however has a limitation.
The limitation is that: _if is the temperature change due to the power flow is too small. The problem turns unfeasible due to the assumption that the cooling system is always needed._

2. The second formulation relaxes the constraints related to the evolution of the temperature of the Battery to be greater than the difference between of the Temperature change given by the power flow and the cooling system.
In other words, it is a worst case scenario.
This formulation incorporates the non-linear behaviour of the cooling system.
In order to do that, a IF-THEN rule representing the step-like function of the cooling system is employed:

If Temperature of the Battery > Ambient temperature THEN 
cooling_on = True.
ELSE 
cooling_on = False.
This can be represented using two variables M1 and M2 that bound the difference between Temperature of the Battery and Ambient temperature.
```latex
\begin{equation*}
-M1 +1 < Temperature of the Battery - Ambient temperature < M2.
\end{equation*}  
```
which can be calculated from the temperature_max and temperature_min conditions.
