# 🧪 Lab Protocols & Standard Operating Procedures (SOPs)

Welcome to the Murali Lab Protocol Repository. This section serves as the centralized directory for all experimental workflows, data analysis pipelines, and equipment operation guidelines used in our laboratory.

!!! note "Mandatory Onboarding Check"
    All lab members and research assistants must review the relevant safety and hardware protocols before conducting experiments with human participants or lab equipment.

---

## 📚 Protocol Directory

### 🧠 Experimental & Hardware SOPs
* **[EEG Acquisition Setup](eeg_setup.md)**: Cap fitting ($10\text{-}20$ system), impedance optimization, amplifier calibration, and session setup.
* **[Equipment Sanitization & Care](sanitization.md)**: Cleaning procedures for gel caps, electrodes, and participant preparation areas.

### 💻 Signal Processing & Computing Pipelines
* **[Data Analysis Pipeline](data_analysis.md)**: Raw data ingestion, artifact rejection (ICA), bandpass filtering, and feature extraction.
* **[Data Organization & Backups](data_management.md)**: Directory naming conventions, server storage structures, and Git version control workflows.

### 🛡️ Ethics, Safety & Administration
* **[Participant Screening & Consent](participant_ethics.md)**: Human subject ethics compliance, consent form administration, and anonymization standards.
* **[Emergency & Safety Procedures](lab_safety.md)**: Incident reporting, first aid locations, and emergency contacts.

---

## 📝 How to Create or Update a Protocol

To maintain high research reproducibility, please follow these steps when adding a new protocol:

1. **Create a new file** inside `docs/protocols/` (e.g., `eyetracking_setup.md`).
2. **Follow the Standard Template Structure**:
   ```markdown
   # Protocol Name

   !!! info "Protocol Details"
       * **Author:** [Your Name]
       * **Last Revised:** [Date]
       * **Equipment Required:** [Equipment list]

   ## 1. Prerequisites & Safety
   ## 2. Step-by-Step Procedure
   ## 3. Data Output & Storage
   ## 4. Common Troubleshooting
