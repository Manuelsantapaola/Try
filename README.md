public class Main {

    public static void main(String[] args) {

        Board board = new Board();

        // Tre pezzi già presenti
        board.placePiece(0, 0, new Piece(1));  // 0001
        board.placePiece(0, 1, new Piece(5));  // 0101
        board.placePiece(0, 2, new Piece(9));  // 1001

        // Pezzo che l'avversario ci ha dato
        Piece pieceToPlace = new Piece(13);    // 1101

        System.out.println("Situazione iniziale:");
        board.printBoard();

        int[] winningMove =
                Solver.findImmediateWin(board, pieceToPlace);

        System.out.println();

        if (winningMove != null) {

            System.out.println(
                    "Mossa vincente trovata: riga "
                    + winningMove[0]
                    + ", colonna "
                    + winningMove[1]
            );

        } else {
            System.out.println("Nessuna vittoria immediata.");
        }
    }
}

public class Solver {

    public static int[] findImmediateWin(Board board, Piece piece) {

        for (int row = 0; row < 4; row++) {

            for (int col = 0; col < 4; col++) {

                if (!board.isEmpty(row, col)) {
                    continue;
                }

                // Provo temporaneamente il pezzo
                board.placePiece(row, col, piece);

                boolean win = WinChecker.hasQuarto(board);

                // Rimetto la board com'era
                board.removePiece(row, col);

                if (win) {
                    return new int[]{row, col};
                }
            }
        }

        return null;
    }
}
