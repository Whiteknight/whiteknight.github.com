Main is treated as a special case because the args to it are stored as a
global in the interp PMC. This shouldn't happen.

Also, we shouldn't do Parrot_pcc_set_sub on the :main inside IMCC. We should
be able to query the location of :main from the packfile,
and call it directly with Parrot_pcc_invoke_sub...

