import java.util.Scanner;

public class Main {

    public static void main(String[] args) {

        Scanner scanner = new Scanner(System.in);

        Board board = new Board();

        // true = pezzo ancora disponibile
        boolean[] availablePieces = new boolean[16];

        for (int i = 0; i < 16; i++) {
            availablePieces[i] = true;
        }

        System.out.println("=== QUARTO ===");
        System.out.println();

        System.out.println("I pezzi vanno da 0 a 15:");
        printAvailablePieces(availablePieces);

        // Il giocatore 1 sceglie il primo pezzo
        System.out.println();
        System.out.println("Giocatore 1: scegli il pezzo da dare al Giocatore 2");

        int selectedPiece =
                choosePiece(scanner, availablePieces);

        // Il pezzo non sarà più disponibile
        availablePieces[selectedPiece] = false;

        int currentPlayer = 2;

        while (true) {

            System.out.println();
            System.out.println("===========================");
            System.out.println("Turno del Giocatore " + currentPlayer);
            System.out.println("===========================");

            Piece pieceToPlace = new Piece(selectedPiece);

            System.out.println(
                    "Devi piazzare il pezzo: "
                    + pieceToPlace
                    + " (" + selectedPiece + ")"
            );

            board.printBoard();

            int row;
            int col;

            while (true) {

                System.out.print("Riga (0-3): ");
                row = scanner.nextInt();

                System.out.print("Colonna (0-3): ");
                col = scanner.nextInt();

                if (
                        row >= 0 && row < 4 &&
                        col >= 0 && col < 4 &&
                        board.isEmpty(row, col)
                ) {
                    break;
                }

                System.out.println(
                        "Posizione non valida. Riprova."
                );
            }

            board.placePiece(row, col, pieceToPlace);

            System.out.println();
            System.out.println("Scacchiera aggiornata:");

            board.printBoard();

            // Controlliamo se il giocatore ha vinto
            if (WinChecker.hasQuarto(board)) {

                System.out.println();
                System.out.println(
                        "QUARTO! Vince il Giocatore "
                        + currentPlayer
                );

                break;
            }

            // Se non ci sono più pezzi la partita è finita
            if (!hasAvailablePieces(availablePieces)) {

                System.out.println();
                System.out.println("Partita terminata in pareggio.");

                break;
            }

            System.out.println();
            System.out.println(
                    "Giocatore " + currentPlayer +
                    ": scegli il pezzo da dare all'avversario"
            );

            printAvailablePieces(availablePieces);

            selectedPiece =
                    choosePiece(scanner, availablePieces);

            availablePieces[selectedPiece] = false;

            // Cambio giocatore
            if (currentPlayer == 1) {
                currentPlayer = 2;
            } else {
                currentPlayer = 1;
            }
        }

        scanner.close();
    }

    public static int choosePiece(
            Scanner scanner,
            boolean[] availablePieces) {

        while (true) {

            System.out.print("Pezzo (0-15): ");

            int piece = scanner.nextInt();

            if (
                    piece >= 0 &&
                    piece < 16 &&
                    availablePieces[piece]
            ) {
                return piece;
            }

            System.out.println(
                    "Pezzo non valido o già utilizzato."
            );
        }
    }

    public static void printAvailablePieces(
            boolean[] availablePieces) {

        for (int i = 0; i < 16; i++) {

            if (availablePieces[i]) {

                String binary =
                        String.format(
                                "%4s",
                                Integer.toBinaryString(i)
                        ).replace(' ', '0');

                System.out.println(
                        i + " -> " + binary
                );
            }
        }
    }

    public static boolean hasAvailablePieces(
            boolean[] availablePieces) {

        for (boolean available : availablePieces) {

            if (available) {
                return true;
            }
        }

        return false;
    }
}
