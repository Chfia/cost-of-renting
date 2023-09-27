<template>
  <div class="equipment-calculator">
    <div v-for="(section, sectionIndex) in equipmentSections" :key="sectionIndex">
      <div class="question">
        <h2>{{ section.name }}</h2>
        <div>
          <div class="equipment" v-for="(item, itemIndex) in section.items" :key="itemIndex">
            <button class="equipment-button" @click="selectEquipment(sectionIndex, item.name, section.selectionType)"
              :class="{  'selected': selectedAnswers[sectionIndex].includes(item.name) }">
              <div class="equipment-option">
                <div class="equipment-image">
                  <img :src="item.imgHref" :alt="item.name" />
                </div>
                <div class="equipment-info">
                  <div class="equipment-name">
                    <strong>{{ item.name }}</strong>
                  </div>
                  <div class="equipment-description">{{ item.description }}</div>
                </div>
              </div>
            </button>

            <button class="equipment-button__desktop" @click="selectEquipment(sectionIndex, item.name, section.selectionType)"
              :class="{  'selected': selectedAnswers[sectionIndex].includes(item.name) }">

              <div class="equipment-option__desktop">
                <div class="equipment-name__desktop">
                  <strong>{{ item.name }}</strong>
                </div>
                <div class="equipment-image__desktop">
                  <img :src="item.imgHref" :alt="item.name" />
                </div>
                <div class="equipment-info__desktop">
                  <div class="equipment-description__desktop">{{ item.description }}</div>
                </div>
              </div>
            </button>
          </div>
        </div>
      </div>
      <div v-if="sectionIndex < equipmentSections.length - 1" class="question-divider"></div>
    </div>
    <div>
      <div class="summary">
        <p class="summary_name">{{ selectedAnswersString }}</p>
        <p class="missing-questions" v-if="!allQuestionsAnswered"
          style="color: black; white-space: pre-line; text-align: left;">{{ missingQuestionsMessage }}</p>
        <div class="line"></div>
        <p class="summary_hours">Work hours: {{ totalWorkHours }}</p>
        <p class="summary_total">Total: {{ showSummary ? totalPrice + ' zł gross' : '-' }}</p>
      </div>
    </div>
  </div>
</template>


<script>
import axios from 'axios';

export default {
  name: 'EquipmentCalculator',
  watch: {
    selectedAnswers: {
      handler: 'calculateCost',
      deep: true,
    },
  },
  data() {
    return {
      equipmentSections: [],
      selectedAnswers: {
        0: [],
        1: [],
        2: [],
        3: [],
        4: [],
      },
      showSummary: false,
      dataJson: null,
      tooltipVisible: false,
      totalPrice: 0,
      totalWorkHours: 0,
    };
  },
  computed: {
    allQuestionsAnswered() {
      return (
        this.selectedAnswers[0].length > 0 &&
        this.selectedAnswers[1].length > 0 &&
        this.selectedAnswers[2].length > 0 &&
        this.selectedAnswers[3].length > 0
      );
    },
    selectedAnswersString() {
      let values = [];
      Object.entries(this.selectedAnswers).forEach((equipment) => {
        if (equipment[1].length > 0) {
          equipment[1].forEach((el) => values.push(el));
        }
      });
      return values.join(' + ');
    },
    missingQuestionsMessage() {
  const missingQuestions = [];
  const questions = [
    'Choose the number of building equipment',
    'Choose the number of building sites',
    'Choose the speed of work',
    'Choose the level of used materials'
  ];

  for (let i = 0; i < questions.length; i++) {
    if (this.selectedAnswers[i].length === 0) {
      missingQuestions.push(`Question ${i + 1} - ${questions[i]}`);
    }
  }

  return `${missingQuestions.join('\n')}.`;
}
   
  },
  mounted() {
    axios
      .get(process.env.BASE_URL + 'data.json')
      .then((response) => {
        this.dataJson = response.data;
        this.equipmentSections = this.dataJson.find(
          (section) => section.type === 'sectionDefinition'
        ).items;
      })
      .catch((error) => {
        console.error('Error loading data:', error);
      });
  },
  methods: {
    selectEquipment(sectionIndex, itemName, selectionType) {
      if (selectionType === 'single') {
        if (this.selectedAnswers[sectionIndex].includes(itemName)) {
          const index = this.selectedAnswers[sectionIndex].indexOf(itemName);
          this.selectedAnswers[sectionIndex].splice(index, 1);
        } else {
          this.selectedAnswers[sectionIndex] = [itemName];
        }
      } else if (selectionType === 'multi') {
        if (this.selectedAnswers[sectionIndex].includes(itemName)) {
          const index = this.selectedAnswers[sectionIndex].indexOf(itemName);
          this.selectedAnswers[sectionIndex].splice(index, 1);
        } else {
          this.selectedAnswers[sectionIndex].push(itemName);
        }
      }
    },
    calculateCost() {
      this.totalPrice = 0;

      const multiplier = this.calculateMultipliers();
      const multipliedBuildingDays = this.calculateMultipliedBuildingDays(multiplier);
      const activeWorkingHours = multipliedBuildingDays * 8;
      const workdayPrice = this.getWorkdayPrice();

      this.totalPrice = activeWorkingHours * workdayPrice;
      this.totalWorkHours = activeWorkingHours;
      this.showSummary = true;
    },
    calculateMultipliers() {
      const buildingEquipment = this.selectedAnswers[0];
      let excavatorCraneMultiplier = 0;
      let rollerMultiplier = 0;
      let speedMultiplier = 0;
      if (buildingEquipment.includes('Excavator')) {
        excavatorCraneMultiplier += 1;
      }
      if (buildingEquipment.includes('Crane')) {
        excavatorCraneMultiplier += 1;
      }
      if (buildingEquipment.includes('Excavator') && buildingEquipment.includes('Crane')) {
        excavatorCraneMultiplier *= 1.2;
      }
      if (this.dataJson && this.selectedAnswers[2][0]) {
        const speedSectionDefinition = this.dataJson[1].items[2].items;
        const selectedSpeed = speedSectionDefinition.find((el) => el.name === this.selectedAnswers[2][0]);
        if (selectedSpeed && selectedSpeed.operationsIfEnabled && selectedSpeed.operationsIfEnabled[0]) {
          speedMultiplier += selectedSpeed.operationsIfEnabled[0].number;
        }
      }

      return excavatorCraneMultiplier + rollerMultiplier + speedMultiplier;
    },
    calculateMultipliedBuildingDays(multiplier) {
      const buildingSitesDefinition = this.dataJson[1].items[1].items;
      const materialsDefinition = this.dataJson[1].items[3].items;
      const additionalFacilitiesDefinition = this.dataJson[1].items[4].items;
      const buildingDays = this.calculateBuildingDays(buildingSitesDefinition, materialsDefinition, additionalFacilitiesDefinition);

      return buildingDays * multiplier;
    },
    calculateBuildingDays(buildingSites, materials, additionalFacilities) {
      let buildingSiteDays = 0;
      let materialDays = 0;

      if (this.selectedAnswers[1][0]) {
        const selectedBuildingSite = buildingSites.find((el) => el.name === this.selectedAnswers[1][0]);
        if (selectedBuildingSite && selectedBuildingSite.operationsIfEnabled && selectedBuildingSite.operationsIfEnabled[0]) {
          buildingSiteDays = selectedBuildingSite.operationsIfEnabled[0].number;
        }
      }

      if (this.selectedAnswers[3][0]) {
        const selectedMaterial = materials.find((el) => el.name === this.selectedAnswers[3][0]);
        if (selectedMaterial && selectedMaterial.operationsIfEnabled && selectedMaterial.operationsIfEnabled[0]) {
          materialDays = selectedMaterial.operationsIfEnabled[0].number;
        }
      }

      let additionalFacilitiesDays = 0;

      additionalFacilities.filter((el) => {
        if (this.selectedAnswers[4].includes(el.name)) {
          return el;
        }
      }).forEach((facility) => {
        additionalFacilitiesDays += facility.operationsIfEnabled[0].number;
      });

      return buildingSiteDays + materialDays + additionalFacilitiesDays;
    },
    getWorkdayPrice() {
      return this.dataJson.filter((data) => {
        return data.type === 'valuesDefinition';
      })[0].items.filter((item) => {
        return item.id === 'workday_price';
      })[0].defaultValue;
    },
  },
};
</script>





<style scoped>
.equipment-button.selected {
  background-color: #a4eba6;
}

.equipment-button__desktop.selected {
  background-color: #a4eba6;
}

.equipment-button {
  width: 100%;
  padding: 0;
  margin: 5px;
  background-color: #ffffff;
  border: none;
  border-radius: 8px;
  cursor: pointer;
  transition: background-color 0.5s;
}

.equipment-button__desktop {
  display: none;
}


.equipment-button:hover {
  background-color: #a4eba6;
}

.equipment-option {
  display: flex;
  align-items: center;
  padding: 15px 20px;
  transition: background-color 0.3s;

 }
 .equipment-description {
  text-align: left;
} 

.equipment-image {
  width: 40px;
  height: 40px;
  min-width: 40px;
  min-height: 40px;
  border-radius: 50%;
  overflow: hidden;
  margin-right: 10px;
}


.equipment-image img {
  width: 100%;
  height: 100%;
  object-fit: cover;

}

.equipment-name {
  flex: 1;
  text-align: left;
}

.question {
  background-color: #f44336;
  padding: 10px;
  margin-bottom: 10px;
  border-radius: 5px;
  color: white;
  padding: 12%;
  z-index: 1;
}


.question-divider {
  height: 10px;
}

.summary {
  background-color: #f44336;
  padding: 20px;
  color: #ffffff;
  margin-top: 10px;
}

.summary .summary_name {
  font-size: 20px;
}

.summary .summary_hours {
  font-size: 15px;
  text-align: left;
}

.summary .summary_total {
  font-size: 25px;
  text-align: left;
}

.summary .line {
  background-color: #ffffff;
  width: 100%;
  height: 2px;
}

@media (min-width: 768px) {

  .question {
  padding: 10px;
  margin: 5px 12%;
  z-index: 1;
}

  .equipment {
    display: inline-block;

  }
  .question:not(:last-child) .equipment:nth-child(2) {
  margin: 0 20%;
}

.summary {
  margin: 0 12%;
  border-radius: 5px;
}
  .equipment-button {
    display: none;
    cursor: pointer;
  }

  .equipment-button__desktop {
    display: flex;
    flex-direction: row;
    align-items: center;
    border-radius: 50%;
    padding: 4px;
    color: white;
    border: none;
    background-color: #f44336;
    transition: background-color 0.5s;
    position: relative;
    margin: 45px 2px 2px 15px;
  }

  .equipment-button__desktop.selected .equipment-description__desktop {
    display: block;

  }

  .equipment-button__desktop .equipment-option__desktop {
    display: flex;
    align-items: center;
    justify-content: center;
  }

  .equipment-button__desktop .equipment-image__desktop {
    width: 80px;
    height: 80px;
    border-radius: 50%;
    overflow: hidden;
    position: relative;
    transition: transform 0.3s ease;
  }


  .equipment-button__desktop .equipment-image__desktop img {
    width: 100%;
    height: 100%;
    object-fit: cover;
  }

  .equipment-name__desktop {
    text-align: center;
    position: absolute;
    top: -30px;
    left: 0;
    right: 0;
    z-index: 999;

  }

  .equipment-button__desktop .equipment-option__desktop:hover .equipment-name__desktop {
    display: block;
  }

  .equipment-button__desktop .equipment-description__desktop {
    display: none;
    width: 400px;
    position: absolute;
    background-color: #696868;
    padding: 10px;
    border-radius: 5px;
    top: 95px;
    left: 85%;
    transform: translateX(-50%);
    z-index: 99;
  }

  .equipment-button__desktop:hover .equipment-description__desktop {
    display: block;
  }

  .equipment-button__desktop .equipment-description__desktop::before {
    content: '';
    position: absolute;
    width: 0;
    height: 0;
    border-left: 10px solid transparent;
    border-right: 10px solid transparent;
    border-top: 10px solid #696868;
    top: -9px;
    left: 40%;
    transform: translateX(-50%);
    transform: rotate(180deg);
  }

  .equipment-button__desktop.selected .equipment-description__desktop::before {
    display: none;
  }

  .equipment-button__desktop.selected .equipment-description__desktop {
    display: none;
  }

}
</style>