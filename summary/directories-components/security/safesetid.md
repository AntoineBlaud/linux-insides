# Safesetid

SafeSetID is an LSM module that gates the setid family of syscalls to restrict UID/GID transitions from a given UID/GID to only those approved by a system-wide allowlist. These restrictions also prohibit the given UIDs/GIDs from obtaining auxiliary privileges associated with CAP\_SET{U/G}ID, such as allowing a user to set up user namespace UID/GID mappings.
