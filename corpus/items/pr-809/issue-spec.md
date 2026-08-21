# Add Date Format Settings and Tooltips for Modified / Created Date Column

<!-- Please only use this template for submitting common issues -->

### Description:
This issue proposes updating the `dateColumn` component to handle flexible, localized, and smart date display formats based on a new user preference, while ensuring maximum data visibility via tooltips.

The proposed updates include:
- **Settings Option:** Add a "Date Display Format" option in the Settings tab with two choices:
  1. `Relative` (e.g., "about 1 month ago")
  2. `Exact Date` (e.g., "Jul 25" or "2025-12-01")
- **Year Omission (for Exact Date):** If the date falls within the current calendar year, omit the year to reduce visual noise (e.g., show "Jul 25" instead of "Jul 25, 2026").
- **Localization (i18n):** Automatically format the "Exact Date" based on the user's current locale/language setting (e.g., "Jul 25, 2026" for en-US, "2026년 7월 25일" for ko-KR) using `Intl.DateTimeFormat` or `date-fns` locales.
- *Tooltip:** Regardless of the selected format, always provide the full, localized `Date + Time` (e.g., "Jul 25, 2026, 3:30 PM") as a tooltip (`title` attribute) on hover.

Additional Checklist:
- **Null & Exception Handling:** Check whether a fallback UI (e.g., `—`) exists for invalid, `null`, or `undefined` date values. If not, implement exception handling.
- **Absolute Sorting:** Ensure the column sorting function consistently compares the raw absolute date strings, independent of the UI display format.

Example(Github):
<img width="274" height="121" alt="Image" src="https://github.com/user-attachments/assets/c5f10051-c0aa-43ab-9424-343e0c54dc47" />

### Why:
- **Simplicity vs. Detail:** Providing two clear display options (`Relative` vs `Exact Date`) keeps the UI simple, while the universal tooltip ensures power users can always access the exact timestamp.
- **Cleaner UI & Native Feel:** Omitting the current year reduces visual clutter, and localizing the date format provides a more native, user-friendly experience for international users.