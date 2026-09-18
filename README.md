import json
import sys
from pathlib import Path

# ============================================================
# AES library
# ============================================================

try:
    from Cryptodome.Cipher import AES
    from Cryptodome.Util.Padding import unpad
except ImportError:
    try:
        from Crypto.Cipher import AES
        from Crypto.Util.Padding import unpad
    except ImportError:
        print(
            "ERROR: Cryptodome / PyCryptodome is not installed.\n"
            "This script will NOT install anything automatically."
        )
        sys.exit(1)


# ============================================================
# DBeaver legacy credential encryption key
# DBeaver 23.x-era credentials-config.json
# ============================================================

DBEAVER_KEY = bytes.fromhex(
    "babb4a9f774ab853c96c2d653dfe544a"
)


def load_json_file(path):
    """Load a normal JSON file."""
    with open(path, "r", encoding="utf-8-sig") as f:
        return json.load(f)


def decrypt_credentials(path):
    """
    Decrypt DBeaver credentials-config.json.

    File layout:
        first 16 bytes  = AES-CBC IV
        remaining bytes = AES-CBC ciphertext
    """

    data = path.read_bytes()

    if not data:
        raise ValueError("credentials-config.json is empty")

    # Some exports may already contain plain JSON.
    stripped = data.lstrip()
    if stripped.startswith(b"{"):
        print("[+] credentials-config.json is already plain JSON")
        return json.loads(data.decode("utf-8-sig"))

    if len(data) < 32:
        raise ValueError(
            "credentials-config.json is too short to be a valid "
            "DBeaver encrypted credentials file"
        )

    if len(data) % AES.block_size != 0:
        raise ValueError(
            "Encrypted file size is not a multiple of AES block size "
            f"(size={len(data)})"
        )

    iv = data[:16]
    ciphertext = data[16:]

    cipher = AES.new(
        DBEAVER_KEY,
        AES.MODE_CBC,
        iv=iv
    )

    padded_plaintext = cipher.decrypt(ciphertext)

    try:
        plaintext = unpad(
            padded_plaintext,
            AES.block_size
        )
    except ValueError as e:
        raise ValueError(
            "AES decryption completed but PKCS#7 padding is invalid. "
            "This usually means this credentials file uses a different "
            "encryption mechanism or secure-storage configuration."
        ) from e

    try:
        text = plaintext.decode("utf-8-sig")
    except UnicodeDecodeError as e:
        raise ValueError(
            "Decrypted data is not valid UTF-8"
        ) from e

    try:
        return json.loads(text)
    except json.JSONDecodeError as e:
        raise ValueError(
            "Decryption produced text, but it is not valid JSON"
        ) from e


def interesting_credentials(section):
    """
    Extract credential-looking values from a credentials section.

    Examples:
        user
        password
        username
        privateKeyPassword
        jumpServer0.password
    """

    result = {}

    if not isinstance(section, dict):
        return result

    for key, value in section.items():
        key_lower = key.lower()

        if (
            "password" in key_lower
            or key_lower in ("user", "username")
        ):
            result[key] = value

    return result


def print_credential_sections(credential_entry):
    """
    Print #connection and other saved credential sections,
    such as SSH tunnel credentials.
    """

    found = False

    if not isinstance(credential_entry, dict):
        return False

    for section_name, section_value in credential_entry.items():

        if not isinstance(section_value, dict):
            continue

        values = interesting_credentials(section_value)

        if not values:
            continue

        found = True

        print(f"Credential section: {section_name}")

        for key, value in values.items():
            print(f"  {key}: {value}")

    # Handle unusual format where credentials are directly
    # under the connection ID instead of inside #connection.
    direct_values = interesting_credentials(credential_entry)

    if direct_values:
        found = True

        print("Credential section: <root>")

        for key, value in direct_values.items():
            print(f"  {key}: {value}")

    return found


def main():

    base_dir = Path(__file__).resolve().parent

    data_sources_path = base_dir / "data-sources.json"
    credentials_path = base_dir / "credentials-config.json"

    print(f"Directory: {base_dir}")
    print()

    # --------------------------------------------------------
    # Check files
    # --------------------------------------------------------

    if not data_sources_path.exists():
        print(
            f"ERROR: Cannot find:\n"
            f"{data_sources_path}"
        )
        sys.exit(1)

    if not credentials_path.exists():
        print(
            f"ERROR: Cannot find:\n"
            f"{credentials_path}"
        )
        sys.exit(1)

    # --------------------------------------------------------
    # Load normal connection definitions
    # --------------------------------------------------------

    try:
        data_sources = load_json_file(
            data_sources_path
        )
    except Exception as e:
        print(
            "ERROR reading data-sources.json:"
        )
        print(type(e).__name__ + ":", e)
        sys.exit(1)

    # --------------------------------------------------------
    # Decrypt credentials
    # --------------------------------------------------------

    try:
        credentials = decrypt_credentials(
            credentials_path
        )
    except Exception as e:
        print(
            "ERROR decrypting credentials-config.json:"
        )
        print(type(e).__name__ + ":", e)
        sys.exit(1)

    print("[+] credentials-config.json decrypted successfully")
    print()

    connections = data_sources.get(
        "connections",
        {}
    )

    if not isinstance(connections, dict):
        print(
            "ERROR: data-sources.json does not contain "
            "a valid 'connections' object."
        )
        sys.exit(1)

    print(
        f"[+] Connections in data-sources.json: "
        f"{len(connections)}"
    )

    if isinstance(credentials, dict):
        print(
            f"[+] Entries in credentials-config.json: "
            f"{len(credentials)}"
        )

    print()
    print("=" * 70)

    # --------------------------------------------------------
    # Correlate connections with credentials
    # --------------------------------------------------------

    for connection_id, connection in connections.items():

        if not isinstance(connection, dict):
            continue

        configuration = connection.get(
            "configuration",
            {}
        )

        if not isinstance(configuration, dict):
            configuration = {}

        name = connection.get(
            "name",
            ""
        )

        provider = connection.get(
            "provider",
            ""
        )

        driver = connection.get(
            "driver",
            ""
        )

        host = configuration.get(
            "host",
            ""
        )

        port = configuration.get(
            "port",
            ""
        )

        database = configuration.get(
            "database",
            ""
        )

        url = configuration.get(
            "url",
            ""
        )

        save_password = connection.get(
            "save-password",
            ""
        )

        print(f"Name:       {name}")
        print(f"ID:         {connection_id}")
        print(f"Provider:   {provider}")
        print(f"Driver:     {driver}")
        print(f"Host:       {host}")
        print(f"Port:       {port}")
        print(f"Database:   {database}")

        if url:
            print(f"URL:        {url}")

        print(f"Save pass:  {save_password}")
        print()

        credential_entry = None

        if isinstance(credentials, dict):
            credential_entry = credentials.get(
                connection_id
            )

        if credential_entry is None:
            print(
                "Credentials: <no matching saved credentials>"
            )
        else:
            found = print_credential_sections(
                credential_entry
            )

            if not found:
                print(
                    "Credentials entry exists, "
                    "but no user/password fields were found."
                )

        print("=" * 70)

    # --------------------------------------------------------
    # Show orphan credential entries
    # --------------------------------------------------------

    if isinstance(credentials, dict):

        orphan_ids = [
            connection_id
            for connection_id in credentials.keys()
            if connection_id not in connections
        ]

        if orphan_ids:
            print()
            print(
                "Saved credential entries with no matching "
                "data-sources.json connection:"
            )
            print()

            for connection_id in orphan_ids:
                print(f"ID: {connection_id}")

                print_credential_sections(
                    credentials[connection_id]
                )

                print("-" * 70)


if __name__ == "__main__":
    main()