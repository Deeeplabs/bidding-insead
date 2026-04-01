## ADDED Requirements

### Requirement: Campaign card modules row supports horizontal scrolling
The campaign card in the Admin Dashboard Campaign modules row SHALL support horizontal scrolling when the number of bidding rounds causes content to exceed the visible width of the card.

#### Scenario: Many bidding rounds overflow the card width
- **WHEN** a campaign has enough bidding rounds that their combined width exceeds the campaign card width
- **THEN** a horizontal scrollbar SHALL appear on the campaign modules row
- **THEN** the user SHALL be able to scroll horizontally to see all bidding rounds and their phases
- **THEN** the overall page layout and other card sections (header, stats boxes, dates) SHALL remain unaffected

#### Scenario: Few bidding rounds fit within card width
- **WHEN** a campaign has few enough bidding rounds that their combined width fits within the campaign card
- **THEN** no scrollbar SHALL be visible on the campaign modules row
- **THEN** the layout SHALL appear identical to the current behaviour

#### Scenario: All phases remain accessible via scroll
- **WHEN** a user scrolls horizontally within the campaign modules row
- **THEN** every bidding round column and its phases SHALL be reachable and clickable
- **THEN** clicking a phase SHALL navigate to the detail-bidding page as before
