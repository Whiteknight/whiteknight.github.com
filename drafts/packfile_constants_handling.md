Using :immediate subs to add constants to packfiles is nice semantically, but
the implementation is dirty

PackfileConstantPMC PMC type
Packfile PMC type

$P0 = new ['Packfile']
$P1 = $P0.add_new_constant($P99) #whatever
$P0.add_new_sub(sub, ... args ... $P1 ...)

In add_new_sub we specify each argument as either being a PackfileRegisterPMC
or a PackfileConstantPMC, and it generates the bytecode accordingly.
