#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <ctype.h>

#define MAX_INSTRUCTIONS 100
#define MAX_LABELS 50
#define MAX_VARIABLES 50
#define MAX_LINE_LENGTH 100
#define CODE_START 0x00000000
#define DATA_START 0x10000000
#define HEAP_START 0x10008000
#define STACK_START 0x7FFFFFFC

// Structure for instruction encoding
typedef struct {
    char *mnemonic;
    char *opcode;
    char *funct3;
    char *funct7;
} InstructionEncoding;

// R, I, S, SB, U, UJ Formats
InstructionEncoding r_format[] = {
    {"add", "0110011", "000", "0000000"},
    {"and", "0110011", "111", "0000000"},
    {"or", "0110011", "110", "0000000"},
    {"sll", "0110011", "001", "0000000"},
    {"slt", "0110011", "010", "0000000"},
    {"sra", "0110011", "101", "0100000"},
    {"srl", "0110011", "101", "0000000"},
    {"sub", "0110011", "000", "0100000"},
    {"xor", "0110011", "100", "0000000"},
    {"mul", "0110011", "000", "0000001"},
    {"div", "0110011", "100", "0000001"},
    {"rem", "0110011", "110", "0000001"}
};

InstructionEncoding i_format[] = {
    {"addi", "0010011", "000", NULL},
    {"andi", "0010011", "111", NULL},
    {"ori", "0010011", "110", NULL},
    {"lb", "0000011", "000", NULL},
    {"ld", "0000011", "011", NULL},
    {"lh", "0000011", "001", NULL},
    {"lw", "0000011", "010", NULL},
    {"jalr", "1100111", "000", NULL}
};

InstructionEncoding s_format[] = {
    {"sb", "0100011", "000", NULL},
    {"sw", "0100011", "010", NULL},
    {"sd", "0100011", "011", NULL},
    {"sh", "0100011", "001", NULL}
};

InstructionEncoding sb_format[] = {
    {"beq", "1100011", "000", NULL},
    {"bne", "1100011", "001", NULL},
    {"bge", "1100011", "101", NULL},
    {"blt", "1100011", "100", NULL}
};

InstructionEncoding u_format[] = {
    {"auipc", "0010111", NULL, NULL},
    {"lui", "0110111", NULL, NULL}
};

InstructionEncoding uj_format[] = {
    {"jal", "1101111", NULL, NULL}
};

// Symbol table for labels
typedef struct {
    char label[MAX_LINE_LENGTH];
    int address;
} LabelTable;

LabelTable labels[MAX_LABELS];
int label_count = 0;

// Structure for variable storage
typedef struct {
    char var_name[MAX_LINE_LENGTH];
    int address;
} VariableTable;

VariableTable variables[MAX_VARIABLES];
int var_count = 0;
int data_address = DATA_START;

// Function to get label address
int get_label_address(char *label) {
    for (int i = 0; i < label_count; i++) {
        if (strcmp(label, labels[i].label) == 0) {
            return labels[i].address; // Return address of the label
        }
    }
    return -1; // Return -1 if label not found (error case)
}

// Function to encode an instruction with label handling
char* encode_instruction(char* instruction, char* operand, int address) {
    static char encoded[50];

    // Check if the operand is a label (used in branch and jump instructions)
    int label_addr = get_label_address(operand);
    if (label_addr != -1) {
        int offset = (label_addr - address) / 4;  // Calculate offset for branch instructions
        sprintf(encoded, "LABEL-ADDR: 0x%X OFFSET: %d", label_addr, offset);
        return encoded;
    }

    // Normal instruction encoding
    for (int i = 0; i < sizeof(r_format)/sizeof(r_format[0]); i++) {
        if (strcmp(instruction, r_format[i].mnemonic) == 0) {
            sprintf(encoded, "%s-%s-%s", r_format[i].opcode, r_format[i].funct3, r_format[i].funct7);
            return encoded;
        }
    }
    for (int i = 0; i < sizeof(i_format)/sizeof(i_format[0]); i++) {
        if (strcmp(instruction, i_format[i].mnemonic) == 0) {
            sprintf(encoded, "%s-%s", i_format[i].opcode, i_format[i].funct3);
            return encoded;
        }
    }
    for (int i = 0; i < sizeof(sb_format)/sizeof(sb_format[0]); i++) {
        if (strcmp(instruction, sb_format[i].mnemonic) == 0) {
            sprintf(encoded, "%s-%s OFFSET: %d", sb_format[i].opcode, sb_format[i].funct3, label_addr);
            return encoded;
        }
    }
    for (int i = 0; i < sizeof(uj_format)/sizeof(uj_format[0]); i++) {
        if (strcmp(instruction, uj_format[i].mnemonic) == 0) {
            sprintf(encoded, "%s ADDR: 0x%X", uj_format[i].opcode, label_addr);
            return encoded;
        }
    }

    return "UNKNOWN";
}

// Function to parse assembly file
void parse_asm_file(const char *input_file, const char *output_file) {
    FILE *in = fopen(input_file, "r");
    FILE *out = fopen(output_file, "w");
    if (!in || !out) {
        printf("Error opening files!\n");
        return;
    }
    
    char line[MAX_LINE_LENGTH];
    int address = CODE_START;
    
    while (fgets(line, sizeof(line), in)) {
        char instruction[MAX_LINE_LENGTH], operand[MAX_LINE_LENGTH];
        sscanf(line, "%s %s", instruction, operand);
        
        // Handle labels
        if (strchr(instruction, ':')) {
            instruction[strlen(instruction) - 1] = '\0';
            strcpy(labels[label_count].label, instruction);
            labels[label_count].address = address;
            label_count++;
            continue;
        }

        // Handle variable storage in .data segment
        if (strcmp(instruction, ".data") == 0) {
            address = DATA_START;
            continue;
        }

        // Replace variable names with addresses
        for (int i = 0; i < var_count; i++) {
            if (strstr(operand, variables[i].var_name)) {
                sprintf(operand, "0x%X", variables[i].address);
            }
        }

        fprintf(out, "0x%X 0x%s , %s %s # %s\n", 
                address, encode_instruction(instruction, operand, address), instruction, operand, encode_instruction(instruction, operand, address));
        
        address += 4;
    }
    
    fclose(in);
    fclose(out);
}

int main() {
    parse_asm_file("input.asm", "output.mc");
    return 0;
}
