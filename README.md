# image_process.py
import os
import glob
import pandas as pd

# Define paths
metadata_file = "/project/samc/battus_compvis/231017_Battus_philenor_polydamas_FLMNH.xlsx"
base_images_root = "/project/samc/battus_compvis/UFLMH_data"
output_file = "/project/samc/battus_compvis/battus/mothra/data/metadata/eval_image_paths.txt"
missing_output_file = "/project/samc/battus_compvis/battus/mothra/data/metadata/eval_image_missing.txt"

def find_image_path(base_dir, genus_species, folder, image_number):
    """
    Searches all timestamped subfolders in base_dir for the correct image.
    Returns the full path if found, otherwise None.
    """
    search_pattern = os.path.join(base_dir, "UF_museum_data_2023-*/UF_museum_data_2023", genus_species, folder, f"IMG_{image_number}.JPG")
    matching_files = glob.glob(search_pattern)

    if matching_files:
        return matching_files[0]  # Return the first match found
    else:
        return None  # Image not found

# Load the metadata
df = pd.read_excel(metadata_file)

# Check if necessary columns exist
required_columns = ['image_folder', 'genus', 'species', 'imag_numbers']
for col in required_columns:
    if col not in df.columns:
        raise ValueError(f"Missing column in metadata file: {col}")

# Data storage
image_paths = {"image_paths": []}
image_missing = {"image_missing": []}

# Process each row
for index, row in df.iterrows():
    genus_species = f"{row['genus']} {row['species']}"
    folder = row['image_folder'].replace('\\', '/')

    # Extract image numbers (handle cases with extra values)
    image_numbers = row['imag_numbers'].split(",")

    # Ensure there are at least two values, otherwise log an issue
    if len(image_numbers) < 2:
        print(f"⚠️ Skipping row {index}: imag_numbers format invalid -> {row['imag_numbers']}")
        continue

    # Take the first two numbers as dorsal and ventral
    dorsal, ventral = image_numbers[:2]  # Only take the first two values

    # Log if there are extra values beyond dorsal/ventral
    if len(image_numbers) > 2:
        print(f"⚠️ Warning: Extra image numbers found in row {index}: {row['imag_numbers']} - Using only first two values.")

    # Search for correct paths
    image_path_d = find_image_path(base_images_root, genus_species, folder, dorsal)
    image_path_v = find_image_path(base_images_root, genus_species, folder, ventral)

    # Debugging: Print the found paths
    print(f"Row {index}: Dorsal={dorsal}, Ventral={ventral}")
    print(f"Checking paths:\n  - Dorsal: {image_path_d}\n  - Ventral: {image_path_v}")

    # Store results
    if image_path_d:
        image_paths["image_paths"].append(image_path_d)
    else:
        print(f"❌ Missing Dorsal Image: {dorsal}")
        image_missing["image_missing"].append(f"{genus_species}/{folder}/IMG_{dorsal}.JPG")

    if image_path_v:
        image_paths["image_paths"].append(image_path_v)
    else:
        print(f"❌ Missing Ventral Image: {ventral}")
        image_missing["image_missing"].append(f"{genus_species}/{folder}/IMG_{ventral}.JPG")

# Save results
pd.DataFrame(image_paths).to_csv(output_file, index=False, header=False)
pd.DataFrame(image_missing).to_csv(missing_output_file, index=False, header=False)

print(f"\n✅ Successfully created {output_file} with {len(image_paths['image_paths'])} image paths.")
print(f"🚨 Missing images saved in {missing_output_file} with {len(image_missing['image_missing'])} entries.")
