src/embed.c:Parrot_pbc_read uses io_eprintf calls directly and is not easy
to override that behavior without major code duplication.

Parrot_pbc_read uses mmap directly. Should maybe use MappedByteArray PMC?
